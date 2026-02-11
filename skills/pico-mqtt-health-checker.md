# Pico MQTT Health Checker - Skill Definition

**Skill ID**: `pico-mqtt-health-checker`
**Category**: IoT / Monitoring / Health Check
**Version**: 1.0.0
**Created**: 2026-02-07
**Platform**: Linux (Ubuntu 22.04+), Python 3.10+

---

## Overview

W5500-EVB-Pico-PoE等のEthernet接続Picoデバイスに対して、ネットワーク接続+MQTT通信の健全性を一括チェックするスキル。
ping疎通、MQTTブローカー接続、メッセージ受信確認を段階的に実行し、障害箇所を特定する。

**Core Capability**: Picoノードの「生存確認」を自動化。物理層→ネットワーク層→アプリケーション層の順に検証し、どの層で問題が発生しているかを切り分ける。

---

## Use Cases

### 1. 農業ハウス運用監視
- PoEノード（温湿度・CO2・照度）の定期ヘルスチェック
- 電源断・ケーブル断の早期検知
- MQTTメッセージ途絶の検知

### 2. デプロイ後の動作確認
- ファームウェア更新後の接続チェック
- ネットワーク設定変更後の疎通確認
- 新規ノード追加時の受け入れテスト

### 3. 障害切り分け
- ping応答なし → 物理層/ネットワーク層の問題
- ping OK + MQTT失敗 → アプリケーション層の問題
- MQTT接続OK + メッセージなし → ファームウェアの問題

---

## Skill Input

| パラメータ | 型 | デフォルト | 説明 |
|-----------|-----|----------|------|
| `target_ip` | string | （必須） | チェック対象デバイスのIP |
| `broker_host` | string | （必須） | MQTTブローカーのIP |
| `broker_port` | int | `1883` | MQTTポート |
| `device_topic` | string | （必須） | デバイスがPublishするトピック（例: `greenhouse/node1/#`） |
| `ping_count` | int | `3` | pingパケット数 |
| `ping_timeout` | int | `5` | pingタイムアウト（秒） |
| `mqtt_wait` | int | `30` | MQTTメッセージ待ち時間（秒） |
| `device_name` | string | `pico-node` | デバイス名（レポート用） |

---

## Generated Output

### ヘルスチェックスクリプト: `pico_mqtt_health_checker.py`

```python
#!/usr/bin/env python3
"""
Pico MQTT Health Checker
Ethernet接続Picoデバイスのネットワーク+MQTT健全性チェック。

Usage:
    python3 pico_mqtt_health_checker.py --target 192.168.15.13 --broker 192.168.15.14 --topic "greenhouse/node1/#"
    python3 pico_mqtt_health_checker.py --target 192.168.15.13 --broker 192.168.15.14 --topic "greenhouse/+/status" --wait 60
"""

import argparse
import json
import subprocess
import sys
import threading
import time
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum

import paho.mqtt.client as mqtt


class HealthStatus(Enum):
    OK = "OK"
    WARN = "WARN"
    FAIL = "FAIL"
    SKIP = "SKIP"


@dataclass
class CheckResult:
    name: str
    status: HealthStatus
    detail: str
    latency_ms: float = 0.0
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())


class PicoMQTTHealthChecker:
    """Picoノードのヘルスチェッカー"""

    def __init__(self, target_ip: str, broker_host: str,
                 broker_port: int = 1883, device_name: str = "pico-node"):
        self.target_ip = target_ip
        self.broker_host = broker_host
        self.broker_port = broker_port
        self.device_name = device_name
        self.results: list[CheckResult] = []

    def check_ping(self, count: int = 3, timeout: int = 5) -> CheckResult:
        """Layer 1-3: Ping疎通確認"""
        try:
            result = subprocess.run(
                ["ping", "-c", str(count), "-W", str(timeout), self.target_ip],
                capture_output=True, text=True, timeout=timeout * count + 5
            )

            if result.returncode == 0:
                # 遅延を抽出
                lines = result.stdout.split("\n")
                for line in lines:
                    if "avg" in line:
                        # rtt min/avg/max/mdev = 0.123/0.456/0.789/0.012 ms
                        parts = line.split("=")[-1].strip().split("/")
                        avg_ms = float(parts[1])
                        status = HealthStatus.OK if avg_ms < 100 else HealthStatus.WARN
                        r = CheckResult("ping", status,
                                        f"{count}/{count} packets, avg={avg_ms:.1f}ms",
                                        latency_ms=avg_ms)
                        self.results.append(r)
                        return r

                r = CheckResult("ping", HealthStatus.OK,
                                f"ping success ({count} packets)")
                self.results.append(r)
                return r
            else:
                r = CheckResult("ping", HealthStatus.FAIL,
                                f"ping failed: {result.stderr.strip()[:100]}")
                self.results.append(r)
                return r

        except subprocess.TimeoutExpired:
            r = CheckResult("ping", HealthStatus.FAIL,
                            f"ping timeout ({timeout}s)")
            self.results.append(r)
            return r

    def check_broker_connection(self) -> CheckResult:
        """Layer 4-5: MQTTブローカー接続確認"""
        client = mqtt.Client(
            client_id=f"health-{int(time.time())}",
            callback_api_version=mqtt.CallbackAPIVersion.VERSION2,
        )
        connected = threading.Event()

        def on_connect(client, userdata, flags, reason_code, properties):
            connected.set()

        client.on_connect = on_connect

        try:
            start = time.monotonic()
            client.connect(self.broker_host, self.broker_port, keepalive=10)
            client.loop_start()

            if connected.wait(timeout=10):
                latency = (time.monotonic() - start) * 1000
                client.disconnect()
                client.loop_stop()
                r = CheckResult("broker_connection", HealthStatus.OK,
                                f"Connected to {self.broker_host}:{self.broker_port}",
                                latency_ms=latency)
            else:
                client.loop_stop()
                r = CheckResult("broker_connection", HealthStatus.FAIL,
                                "Connection timeout (10s)")

            self.results.append(r)
            return r

        except Exception as e:
            r = CheckResult("broker_connection", HealthStatus.FAIL, str(e))
            self.results.append(r)
            return r

    def check_device_messages(self, topic: str,
                              wait_seconds: int = 30) -> CheckResult:
        """Layer 7: デバイスからのMQTTメッセージ受信確認"""
        client = mqtt.Client(
            client_id=f"health-sub-{int(time.time())}",
            callback_api_version=mqtt.CallbackAPIVersion.VERSION2,
        )
        messages = []
        first_msg = threading.Event()

        def on_message(client, userdata, msg):
            messages.append({
                "topic": msg.topic,
                "payload": msg.payload.decode("utf-8", errors="replace")[:200],
                "qos": msg.qos,
                "time": datetime.now().isoformat(),
            })
            first_msg.set()

        client.on_message = on_message

        try:
            client.connect(self.broker_host, self.broker_port, keepalive=wait_seconds + 10)
            client.subscribe(topic, qos=0)
            client.loop_start()

            start = time.monotonic()
            received = first_msg.wait(timeout=wait_seconds)
            elapsed = (time.monotonic() - start) * 1000

            client.disconnect()
            client.loop_stop()

            if received:
                r = CheckResult(
                    "device_messages", HealthStatus.OK,
                    f"Received {len(messages)} msg(s) on '{topic}' "
                    f"(first in {elapsed:.0f}ms)",
                    latency_ms=elapsed
                )
            else:
                r = CheckResult(
                    "device_messages", HealthStatus.FAIL,
                    f"No messages on '{topic}' after {wait_seconds}s"
                )

            self.results.append(r)
            return r

        except Exception as e:
            r = CheckResult("device_messages", HealthStatus.FAIL, str(e))
            self.results.append(r)
            return r

    def run_all(self, topic: str, ping_count: int = 3,
                ping_timeout: int = 5,
                mqtt_wait: int = 30) -> dict:
        """全チェック実行（段階的）"""
        print(f"=== Pico MQTT Health Check ===")
        print(f"Device: {self.device_name} ({self.target_ip})")
        print(f"Broker: {self.broker_host}:{self.broker_port}")
        print(f"Topic: {topic}")
        print()

        # Step 1: Ping
        print("[1/3] Ping check...")
        ping_result = self.check_ping(ping_count, ping_timeout)
        self._print_result(ping_result)

        if ping_result.status == HealthStatus.FAIL:
            print("\n⚠️ Ping failed - device may be offline or unreachable")
            # ping失敗でも残りのテストは続行（ブローカー側の問題かもしれない）

        # Step 2: Broker接続
        print("[2/3] Broker connection check...")
        broker_result = self.check_broker_connection()
        self._print_result(broker_result)

        if broker_result.status == HealthStatus.FAIL:
            print("\n⚠️ Broker connection failed - skipping message check")
            skip = CheckResult("device_messages", HealthStatus.SKIP,
                               "Skipped (broker unreachable)")
            self.results.append(skip)
            return self._summary()

        # Step 3: メッセージ受信
        print(f"[3/3] Waiting for device messages ({mqtt_wait}s)...")
        self.check_device_messages(topic, mqtt_wait)
        self._print_result(self.results[-1])

        return self._summary()

    @staticmethod
    def _print_result(r: CheckResult):
        """結果表示"""
        icons = {
            HealthStatus.OK: "✅",
            HealthStatus.WARN: "⚠️",
            HealthStatus.FAIL: "❌",
            HealthStatus.SKIP: "⏭️",
        }
        lat = f" ({r.latency_ms:.1f}ms)" if r.latency_ms > 0 else ""
        print(f"  {icons[r.status]} {r.name}: {r.detail}{lat}")

    def _summary(self) -> dict:
        """結果サマリ"""
        ok_count = sum(1 for r in self.results if r.status == HealthStatus.OK)
        total = sum(1 for r in self.results if r.status != HealthStatus.SKIP)
        has_fail = any(r.status == HealthStatus.FAIL for r in self.results)

        overall = "HEALTHY" if not has_fail else "UNHEALTHY"

        print(f"\n=== Overall: {overall} ({ok_count}/{total} passed) ===")

        # 障害切り分けガイド
        if has_fail:
            print("\n📋 Diagnosis:")
            for r in self.results:
                if r.status == HealthStatus.FAIL:
                    if r.name == "ping":
                        print("  → Device unreachable: check cable/power/IP config")
                    elif r.name == "broker_connection":
                        print("  → Broker issue: check Mosquitto service/firewall")
                    elif r.name == "device_messages":
                        print("  → No messages: check firmware/MQTT config on device")

        return {
            "overall": overall,
            "device": self.device_name,
            "target_ip": self.target_ip,
            "results": [
                {
                    "name": r.name,
                    "status": r.status.value,
                    "detail": r.detail,
                    "latency_ms": r.latency_ms,
                }
                for r in self.results
            ],
        }


def main():
    parser = argparse.ArgumentParser(description="Pico MQTT Health Checker")
    parser.add_argument("--target", required=True, help="Device IP address")
    parser.add_argument("--broker", required=True, help="MQTT broker host/IP")
    parser.add_argument("--port", type=int, default=1883, help="MQTT port")
    parser.add_argument("--topic", required=True, help="Device MQTT topic (wildcards OK)")
    parser.add_argument("--ping-count", type=int, default=3, help="Ping packet count")
    parser.add_argument("--ping-timeout", type=int, default=5, help="Ping timeout (sec)")
    parser.add_argument("--wait", type=int, default=30, help="MQTT message wait (sec)")
    parser.add_argument("--name", default="pico-node", help="Device name for report")
    parser.add_argument("--json", action="store_true", help="Output JSON")
    args = parser.parse_args()

    checker = PicoMQTTHealthChecker(args.target, args.broker, args.port, args.name)
    result = checker.run_all(
        topic=args.topic,
        ping_count=args.ping_count,
        ping_timeout=args.ping_timeout,
        mqtt_wait=args.wait,
    )

    if args.json:
        print(json.dumps(result, indent=2, ensure_ascii=False))

    sys.exit(0 if result["overall"] == "HEALTHY" else 1)


if __name__ == "__main__":
    main()
```

---

## Health Check Layers

```
Layer    Check              Tool          Failure Meaning
──────   ─────────────────  ──────────    ─────────────────────────
L1-3     Ping               ping          Cable/Power/IP issue
L4-5     Broker Connection  paho-mqtt     Broker down/Firewall
L7       Device Messages    paho-mqtt     Firmware/Config issue
```

---

## Multi-Node Batch Check

複数ノードを一括チェックする例:

```bash
#!/bin/bash
# batch_health_check.sh
BROKER="192.168.15.14"
NODES=(
    "192.168.15.13:greenhouse/node1/#:climate-node-1"
    "192.168.15.20:greenhouse/node2/#:drainage-node"
    "192.168.15.21:greenhouse/node3/#:solar-sensor"
)

for node in "${NODES[@]}"; do
    IFS=':' read -r ip topic name <<< "$node"
    echo "=========================================="
    python3 pico_mqtt_health_checker.py \
        --target "$ip" --broker "$BROKER" \
        --topic "$topic" --name "$name" --wait 15
    echo
done
```

---

## Dependencies

| パッケージ | バージョン | 用途 |
|-----------|----------|------|
| `paho-mqtt` | 2.0+ | MQTTクライアント |
| `ping` | (system) | ICMP疎通確認 |

### インストール

```bash
pip install paho-mqtt
```

---

## Troubleshooting

| 症状 | 層 | 原因 | 対処 |
|------|-----|------|------|
| Ping失敗 | L1-3 | ケーブル断/電源断/IP不正 | 物理接続・PoE給電を確認 |
| Ping OK + Broker失敗 | L4-5 | Mosquitto停止/ポートブロック | `systemctl status mosquitto` |
| 全OK + メッセージなし | L7 | ファームウェア停止/トピック不正 | シリアル接続でREPL確認 |
| 高遅延 (>100ms) | L1-3 | ネットワーク輻輳/PoEスイッチ負荷 | ネットワーク機器を確認 |
