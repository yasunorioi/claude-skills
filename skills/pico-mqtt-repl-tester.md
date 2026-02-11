# Pico MQTT REPL Tester - Skill Definition

**Skill ID**: `pico-mqtt-repl-tester`
**Category**: IoT / Testing / Serial Communication
**Version**: 1.0.0
**Created**: 2026-02-07
**Platform**: Linux (Ubuntu 22.04+), Python 3.10+, pyserial

---

## Overview

CircuitPython REPL経由でPico W / W5500-EVB-Pico-PoEのMQTT publish/subscribeをテストするスキル。
pyserialによるシリアル接続で安全にREPLコマンドを送信し、MQTT通信の動作確認を自動化する。

**Core Capability**: シリアルポート経由でCircuitPython REPLにMQTTテストコマンドを送り、結果をパースして構造化レポートを生成する。

**重要**: シリアルデバイスの直接catは**厳禁**（切腹ルール）。必ずpyserialのtimeout付きで操作せよ。

---

## Use Cases

### 1. ファームウェア開発時のMQTTテスト
- ファームウェア書き換え後のMQTT疎通確認
- Publish/Subscribe/LWTの動作検証
- QoSレベルの確認

### 2. ハードウェア検品
- 新規Pico基板のMQTT通信テスト
- W5500 Ethernet接続 + MQTT動作確認
- WiFiモジュールの接続テスト

### 3. 障害切り分け
- MQTT接続失敗時のデバッグ（ネットワーク vs ファームウェア）
- ブローカー接続パラメータの検証
- シリアル接続自体の健全性確認

---

## Skill Input

| パラメータ | 型 | デフォルト | 説明 |
|-----------|-----|----------|------|
| `serial_port` | string | `/dev/ttyACM0` | シリアルポートパス |
| `baud_rate` | int | `115200` | ボーレート |
| `mqtt_broker` | string | （必須） | MQTTブローカーのIPアドレス |
| `mqtt_port` | int | `1883` | MQTTポート |
| `mqtt_topic` | string | `test/pico/repl` | テスト用MQTTトピック |
| `timeout` | int | `10` | シリアルコマンドのタイムアウト（秒） |
| `network_type` | string | `ethernet` | `ethernet` or `wifi` |

---

## Generated Output

### テストスクリプト: `pico_mqtt_repl_test.py`

```python
#!/usr/bin/env python3
"""
Pico MQTT REPL Tester
CircuitPython REPL経由でMQTT publish/subscribeをテスト。

Usage:
    python3 pico_mqtt_repl_test.py --port /dev/ttyACM0 --broker 192.168.15.14
    python3 pico_mqtt_repl_test.py --port /dev/ttyACM0 --broker 192.168.15.14 --wifi
"""

import argparse
import json
import sys
import time
from datetime import datetime

import serial


class PicoMQTTREPLTester:
    """CircuitPython REPL経由のMQTTテスター"""

    def __init__(self, port: str, baud: int = 115200, timeout: int = 10):
        self.port = port
        self.baud = baud
        self.timeout = timeout
        self.ser = None
        self.results = []

    def connect(self) -> bool:
        """シリアル接続（timeout必須 - 切腹ルール遵守）"""
        try:
            self.ser = serial.Serial(
                port=self.port,
                baudrate=self.baud,
                timeout=self.timeout,      # 読み取りタイムアウト（必須）
                write_timeout=self.timeout  # 書き込みタイムアウト（必須）
            )
            time.sleep(0.5)  # 接続安定化待ち
            return True
        except serial.SerialException as e:
            self._record("serial_connect", False, str(e))
            return False

    def disconnect(self):
        """シリアル切断"""
        if self.ser and self.ser.is_open:
            self.ser.close()

    def send_repl_command(self, cmd: str, wait: float = 2.0) -> str:
        """REPLコマンド送信+応答取得（安全操作）"""
        if not self.ser or not self.ser.is_open:
            return ""

        # Ctrl+C で既存処理を中断
        self.ser.write(b"\x03\x03")
        time.sleep(0.3)
        self.ser.read(self.ser.in_waiting or 1)  # バッファクリア

        # コマンド送信
        self.ser.write(f"{cmd}\r\n".encode("utf-8"))
        time.sleep(wait)

        # 応答読み取り（timeout付きで安全）
        response = ""
        while self.ser.in_waiting:
            chunk = self.ser.read(self.ser.in_waiting)
            try:
                response += chunk.decode("utf-8", errors="replace")
            except Exception:
                response += "[binary data]"
            time.sleep(0.1)

        return response

    def test_repl_alive(self) -> bool:
        """REPL応答テスト"""
        resp = self.send_repl_command("print('REPL_OK')", wait=1.0)
        ok = "REPL_OK" in resp
        self._record("repl_alive", ok, resp.strip()[-100:] if resp else "no response")
        return ok

    def test_mqtt_import(self) -> bool:
        """MQTTライブラリインポートテスト"""
        resp = self.send_repl_command(
            "import adafruit_minimqtt.adafruit_minimqtt as MQTT; print('IMPORT_OK')",
            wait=2.0
        )
        ok = "IMPORT_OK" in resp
        self._record("mqtt_import", ok, resp.strip()[-100:] if resp else "no response")
        return ok

    def test_network_init(self, network_type: str = "ethernet") -> bool:
        """ネットワーク初期化テスト"""
        if network_type == "ethernet":
            commands = [
                "import board, busio, digitalio",
                "from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K",
                "cs = digitalio.DigitalInOut(board.GP17)",
                "spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)",
                "eth = WIZNET5K(spi, cs, is_dhcp=True)",
                "print('IP:', eth.pretty_ip(eth.ip_address))",
            ]
        else:  # wifi
            commands = [
                "import wifi, os",
                "wifi.radio.connect(os.getenv('CIRCUITPY_WIFI_SSID'), os.getenv('CIRCUITPY_WIFI_PASSWORD'))",
                "print('IP:', wifi.radio.ipv4_address)",
            ]

        full_cmd = "; ".join(commands)
        resp = self.send_repl_command(full_cmd, wait=5.0)
        ok = "IP:" in resp
        self._record("network_init", ok, resp.strip()[-200:] if resp else "no response")
        return ok

    def test_mqtt_publish(self, broker: str, port: int = 1883,
                          topic: str = "test/pico/repl") -> bool:
        """MQTT Publishテスト"""
        timestamp = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")
        payload = json.dumps({"test": True, "ts": timestamp, "src": "repl_tester"})

        commands = [
            "from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool",
            "pool = SocketPool(eth)",
            f"mqtt = MQTT.MQTT(broker='{broker}', port={port}, socket_pool=pool)",
            "mqtt.connect()",
            f"mqtt.publish('{topic}', '{payload}')",
            "print('PUBLISH_OK')",
            "mqtt.disconnect()",
        ]

        full_cmd = "; ".join(commands)
        resp = self.send_repl_command(full_cmd, wait=5.0)
        ok = "PUBLISH_OK" in resp
        self._record("mqtt_publish", ok, resp.strip()[-200:] if resp else "no response")
        return ok

    def test_mqtt_subscribe(self, broker: str, port: int = 1883,
                            topic: str = "test/pico/repl/sub") -> bool:
        """MQTT Subscribeテスト（短時間のみ）"""
        commands = [
            "from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool",
            "pool = SocketPool(eth)",
            f"mqtt2 = MQTT.MQTT(broker='{broker}', port={port}, socket_pool=pool)",
            "received = []",
            "def on_msg(client, topic, message): received.append(message); print('RECV:', message)",
            "mqtt2.on_message = on_msg",
            "mqtt2.connect()",
            f"mqtt2.subscribe('{topic}')",
            "import time; [mqtt2.loop(timeout=1) for _ in range(3)]",
            "print('SUB_OK, received:', len(received))",
            "mqtt2.disconnect()",
        ]

        full_cmd = "; ".join(commands)
        resp = self.send_repl_command(full_cmd, wait=8.0)
        ok = "SUB_OK" in resp
        self._record("mqtt_subscribe", ok, resp.strip()[-200:] if resp else "no response")
        return ok

    def _record(self, test_name: str, passed: bool, detail: str):
        """テスト結果記録"""
        self.results.append({
            "test": test_name,
            "passed": passed,
            "detail": detail,
            "timestamp": datetime.now().isoformat(),
        })

    def run_all(self, broker: str, port: int = 1883,
                topic: str = "test/pico/repl",
                network_type: str = "ethernet") -> dict:
        """全テスト実行"""
        print(f"=== Pico MQTT REPL Test ===")
        print(f"Port: {self.port}, Broker: {broker}:{port}")
        print(f"Network: {network_type}, Topic: {topic}")
        print()

        if not self.connect():
            return self._summary()

        try:
            # Step 1: REPL応答
            if not self.test_repl_alive():
                return self._summary()

            # Step 2: MQTTライブラリ
            if not self.test_mqtt_import():
                return self._summary()

            # Step 3: ネットワーク初期化
            if not self.test_network_init(network_type):
                return self._summary()

            # Step 4: MQTT Publish
            self.test_mqtt_publish(broker, port, topic)

            # Step 5: MQTT Subscribe
            self.test_mqtt_subscribe(broker, port, f"{topic}/sub")

        finally:
            self.disconnect()

        return self._summary()

    def _summary(self) -> dict:
        """結果サマリ"""
        passed = sum(1 for r in self.results if r["passed"])
        total = len(self.results)
        print(f"\n=== Results: {passed}/{total} PASSED ===")
        for r in self.results:
            status = "✅" if r["passed"] else "❌"
            print(f"  {status} {r['test']}: {r['detail'][:80]}")
        return {"passed": passed, "total": total, "results": self.results}


def main():
    parser = argparse.ArgumentParser(description="Pico MQTT REPL Tester")
    parser.add_argument("--port", default="/dev/ttyACM0", help="Serial port")
    parser.add_argument("--baud", type=int, default=115200, help="Baud rate")
    parser.add_argument("--broker", required=True, help="MQTT broker IP")
    parser.add_argument("--mqtt-port", type=int, default=1883, help="MQTT port")
    parser.add_argument("--topic", default="test/pico/repl", help="MQTT topic")
    parser.add_argument("--timeout", type=int, default=10, help="Serial timeout (sec)")
    parser.add_argument("--wifi", action="store_true", help="Use WiFi instead of Ethernet")
    args = parser.parse_args()

    tester = PicoMQTTREPLTester(args.port, args.baud, args.timeout)
    result = tester.run_all(
        broker=args.broker,
        port=args.mqtt_port,
        topic=args.topic,
        network_type="wifi" if args.wifi else "ethernet",
    )
    sys.exit(0 if result["passed"] == result["total"] else 1)


if __name__ == "__main__":
    main()
```

---

## Safety Rules

### 切腹ルール（シリアルデバイス安全操作）

| ルール | 説明 |
|--------|------|
| **timeout必須** | `serial.Serial(timeout=N)` を必ず指定。無限待ちは禁止 |
| **cat禁止** | `cat /dev/ttyACMx` は厳禁。バイナリがターミナルを破壊する |
| **バイナリ保護** | `decode("utf-8", errors="replace")` で非UTF-8を安全に処理 |
| **Ctrl+C先行** | コマンド送信前にCtrl+Cで既存処理を停止 |
| **バッファクリア** | 応答読み取り前に受信バッファをクリア |

### 安全なシリアル読み取り方法

```bash
# ✅ 安全: pyserialでtimeout付き（推奨）
python3 -c "import serial; s=serial.Serial('/dev/ttyACM0', timeout=5); print(s.read(1024))"

# ✅ 安全: timeout + xxd でバイナリセーフ
timeout 5 cat /dev/ttyACM0 2>/dev/null | xxd | head

# ✅ 安全: ファイルにリダイレクト
timeout 5 cat /dev/ttyACM0 > /tmp/serial_out.bin 2>/dev/null

# ❌ 危険: 直接catは絶対禁止
cat /dev/ttyACM0   # ← ターミナル破壊の危険
```

---

## Dependencies

| パッケージ | バージョン | 用途 |
|-----------|----------|------|
| `pyserial` | 3.5+ | シリアル通信 |
| `paho-mqtt` | 2.0+ | ブローカー側検証（オプション） |

### インストール

```bash
pip install pyserial paho-mqtt
```

---

## CircuitPython 10.x API Notes

CircuitPython 10.x ではソケットAPIが変更されている。

```python
# ❌ 旧API (9.x) - 動作しない
from adafruit_wiznet5k import adafruit_wiznet5k_socket as socket
MQTT.set_socket(socket, eth)

# ✅ 新API (10.x) - 正しい方法
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool
pool = SocketPool(eth)
mqtt = MQTT.MQTT(broker="...", port=1883, socket_pool=pool)
```

---

## Troubleshooting

| 症状 | 原因 | 対処 |
|------|------|------|
| `SerialException: could not open port` | デバイス未接続 or 権限不足 | `ls /dev/ttyACM*` + `sudo usermod -a -G dialout $USER` |
| REPL無応答 | ファームウェア実行中 | Ctrl+C送信回数を増やす（3回以上） |
| `MQTT connect failed` | ブローカー未起動 or IP不正 | `mosquitto_sub -h <broker> -t '#'` で疎通確認 |
| DHCP timeout | ネットワークケーブル未接続 | 物理接続確認。W5500のLEDが点灯しているか |
| Import error | ライブラリ未インストール | `circup install adafruit_minimqtt adafruit_wiznet5k` |
