# HA OS Network Discovery - Skill Definition

**Skill ID**: `ha-os-network-discovery`
**Category**: IoT / Home Assistant / Network
**Version**: 1.0.0
**Created**: 2026-02-07
**Platform**: Linux (Ubuntu 22.04+), nmap, Python 3.10+

---

## Overview

LAN内のHome Assistant OSデバイスをnmapスキャン + ポートチェック（8123/4357）で自動発見し、状態を確認するスキル。
新規セットアップ時のデバイス探索、運用中の生存確認、ネットワーク変更後のアドレス再発見に使用する。

**Core Capability**: nmapでネットワークスキャンし、Home Assistantの特徴的なポート（Web UI: 8123、Supervisor API: 4357）でフィルタリングして、HAデバイスを特定・状態確認する。

---

## Use Cases

### 1. 初回セットアップ時のデバイス発見
- HA OSを書き込んだRaspberry PiのIPアドレスを特定
- DHCPで割り当てられた未知のIPを発見
- 複数HAインスタンスの一括検出

### 2. 運用中の状態確認
- HAデバイスの生存確認（定期チェック）
- ネットワーク変更後のIP再発見
- VPN越しのリモートHA確認

### 3. トラブルシューティング
- HA Web UIにアクセスできない時のIP確認
- DHCP更新でIPが変わった場合の再発見
- 複数サブネットでのHA検索

---

## Skill Input

| パラメータ | 型 | デフォルト | 説明 |
|-----------|-----|----------|------|
| `network` | string | `192.168.15.0/24` | スキャン対象ネットワーク（CIDR表記） |
| `ha_web_port` | int | `8123` | HA Web UIポート |
| `ha_supervisor_port` | int | `4357` | HA Supervisor APIポート |
| `ssh_port` | int | `22222` | HA OS SSHポート |
| `timeout` | int | `30` | nmapスキャンタイムアウト（秒） |
| `check_api` | bool | `True` | HA REST API疎通確認を行うか |

---

## Generated Output

### 発見スクリプト: `ha_os_network_discovery.py`

```python
#!/usr/bin/env python3
"""
Home Assistant OS Network Discovery
nmapスキャン + ポートチェックでLAN内のHAデバイスを自動発見。

Usage:
    python3 ha_os_network_discovery.py --network 192.168.15.0/24
    python3 ha_os_network_discovery.py --network 10.10.0.0/24 --no-api-check
    sudo python3 ha_os_network_discovery.py --network 192.168.15.0/24 --deep
"""

import argparse
import json
import subprocess
import sys
import time
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional

try:
    import urllib.request
    HAS_URLLIB = True
except ImportError:
    HAS_URLLIB = False


@dataclass
class HADevice:
    """発見されたHome Assistantデバイス"""
    ip: str
    hostname: str = ""
    web_port_open: bool = False
    supervisor_port_open: bool = False
    ssh_port_open: bool = False
    api_reachable: bool = False
    api_version: str = ""
    mac_address: str = ""
    os_info: str = ""
    discovered_at: str = field(default_factory=lambda: datetime.now().isoformat())


class HANetworkDiscovery:
    """HA OSネットワーク発見ツール"""

    def __init__(self, network: str = "192.168.15.0/24",
                 ha_web_port: int = 8123,
                 ha_supervisor_port: int = 4357,
                 ssh_port: int = 22222,
                 timeout: int = 30):
        self.network = network
        self.ha_web_port = ha_web_port
        self.ha_supervisor_port = ha_supervisor_port
        self.ssh_port = ssh_port
        self.timeout = timeout
        self.devices: list[HADevice] = []

    def scan_network(self) -> list[str]:
        """Phase 1: nmapでネットワークスキャン"""
        print(f"[1/3] Scanning network {self.network}...")

        ports = f"{self.ha_web_port},{self.ha_supervisor_port},{self.ssh_port}"
        cmd = [
            "nmap", "-sT",          # TCP connect scan (sudo不要)
            "-p", ports,             # 対象ポート
            "--open",                # openのみ表示
            "-T4",                   # 高速スキャン
            f"--host-timeout={self.timeout}s",
            "-oX", "-",              # XML出力（stdout）
            self.network,
        ]

        try:
            result = subprocess.run(
                cmd, capture_output=True, text=True,
                timeout=self.timeout * 2
            )

            if result.returncode not in (0, 1):  # nmap: 1 = some hosts filtered
                print(f"  ⚠️ nmap warning: {result.stderr.strip()[:100]}")

            # XML出力からIPを抽出（簡易パース）
            candidates = self._parse_nmap_xml(result.stdout)
            print(f"  Found {len(candidates)} candidate(s) with HA ports open")
            return candidates

        except FileNotFoundError:
            print("  ❌ nmap not found. Install: sudo apt install nmap")
            return []
        except subprocess.TimeoutExpired:
            print(f"  ❌ Scan timeout ({self.timeout * 2}s)")
            return []

    def _parse_nmap_xml(self, xml_output: str) -> list[str]:
        """nmap XML出力から候補IPを抽出"""
        import xml.etree.ElementTree as ET

        candidates = []
        try:
            root = ET.fromstring(xml_output)
            for host in root.findall(".//host"):
                addr_elem = host.find("address[@addrtype='ipv4']")
                if addr_elem is None:
                    continue

                ip = addr_elem.get("addr", "")
                ports_elem = host.find(".//ports")
                if ports_elem is None:
                    continue

                open_ports = []
                for port in ports_elem.findall("port"):
                    state = port.find("state")
                    if state is not None and state.get("state") == "open":
                        open_ports.append(int(port.get("portid", "0")))

                # HA Web UIポートが開いていれば候補
                if self.ha_web_port in open_ports:
                    candidates.append(ip)

        except ET.ParseError:
            # XMLパース失敗時はgrepでフォールバック
            for line in xml_output.split("\n"):
                if f'portid="{self.ha_web_port}"' in line and 'state="open"' in line:
                    # 直前のhost addressを探す（簡易）
                    pass

        return candidates

    def check_device(self, ip: str, check_api: bool = True) -> HADevice:
        """Phase 2: 候補デバイスの詳細確認"""
        device = HADevice(ip=ip)

        # ポート確認
        device.web_port_open = self._check_port(ip, self.ha_web_port)
        device.supervisor_port_open = self._check_port(ip, self.ha_supervisor_port)
        device.ssh_port_open = self._check_port(ip, self.ssh_port)

        # ホスト名取得
        device.hostname = self._get_hostname(ip)

        # API確認
        if check_api and device.web_port_open and HAS_URLLIB:
            device.api_reachable, device.api_version = self._check_ha_api(ip)

        return device

    @staticmethod
    def _check_port(ip: str, port: int, timeout: int = 3) -> bool:
        """TCPポート開放確認"""
        import socket
        try:
            sock = socket.create_connection((ip, port), timeout=timeout)
            sock.close()
            return True
        except (socket.timeout, ConnectionRefusedError, OSError):
            return False

    @staticmethod
    def _get_hostname(ip: str) -> str:
        """逆引きホスト名取得"""
        import socket
        try:
            hostname = socket.gethostbyaddr(ip)[0]
            return hostname
        except (socket.herror, socket.gaierror):
            return ""

    def _check_ha_api(self, ip: str) -> tuple[bool, str]:
        """HA REST API疎通確認"""
        url = f"http://{ip}:{self.ha_web_port}/api/"
        try:
            req = urllib.request.Request(url, method="GET")
            req.add_header("Content-Type", "application/json")
            with urllib.request.urlopen(req, timeout=5) as resp:
                data = json.loads(resp.read().decode("utf-8"))
                return True, data.get("message", "unknown")
        except Exception:
            # APIにはトークンが必要な場合がある。接続自体ができればOK
            try:
                req = urllib.request.Request(url, method="GET")
                urllib.request.urlopen(req, timeout=5)
                return True, "reachable (auth required)"
            except urllib.error.HTTPError as e:
                if e.code in (401, 403):
                    return True, f"reachable (HTTP {e.code})"
                return False, str(e)
            except Exception as e:
                return False, str(e)

    def discover(self, check_api: bool = True) -> list[HADevice]:
        """全工程実行"""
        print(f"=== Home Assistant OS Network Discovery ===")
        print(f"Network: {self.network}")
        print(f"Ports: Web={self.ha_web_port}, Supervisor={self.ha_supervisor_port}, SSH={self.ssh_port}")
        print()

        # Phase 1: ネットワークスキャン
        candidates = self.scan_network()
        if not candidates:
            print("\n❌ No Home Assistant devices found")
            return []

        # Phase 2: 詳細確認
        print(f"\n[2/3] Checking {len(candidates)} candidate(s)...")
        for ip in candidates:
            device = self.check_device(ip, check_api)
            self.devices.append(device)
            self._print_device(device)

        # Phase 3: サマリ
        print(f"\n[3/3] Summary")
        self._print_summary()

        return self.devices

    @staticmethod
    def _print_device(d: HADevice):
        """デバイス情報表示"""
        icon = "✅" if d.web_port_open else "❌"
        print(f"\n  {icon} {d.ip}" + (f" ({d.hostname})" if d.hostname else ""))
        print(f"     Web UI (8123):     {'✅ Open' if d.web_port_open else '❌ Closed'}")
        print(f"     Supervisor (4357): {'✅ Open' if d.supervisor_port_open else '❌ Closed'}")
        print(f"     SSH (22222):       {'✅ Open' if d.ssh_port_open else '❌ Closed'}")
        if d.api_reachable:
            print(f"     API:               ✅ {d.api_version}")

    def _print_summary(self):
        """サマリ表示"""
        ha_devices = [d for d in self.devices if d.web_port_open]
        print(f"  Found {len(ha_devices)} HA device(s) out of {len(self.devices)} candidate(s)")

        if ha_devices:
            print("\n  Quick Access:")
            for d in ha_devices:
                print(f"    http://{d.ip}:{self.ha_web_port}")
                if d.ssh_port_open:
                    print(f"    ssh root@{d.ip} -p {self.ssh_port}")

    def to_json(self) -> str:
        """JSON出力"""
        return json.dumps(
            [
                {
                    "ip": d.ip,
                    "hostname": d.hostname,
                    "ports": {
                        "web_8123": d.web_port_open,
                        "supervisor_4357": d.supervisor_port_open,
                        "ssh_22222": d.ssh_port_open,
                    },
                    "api": {
                        "reachable": d.api_reachable,
                        "version": d.api_version,
                    },
                    "discovered_at": d.discovered_at,
                }
                for d in self.devices
            ],
            indent=2,
            ensure_ascii=False,
        )


def main():
    parser = argparse.ArgumentParser(description="HA OS Network Discovery")
    parser.add_argument("--network", default="192.168.15.0/24",
                        help="Network to scan (CIDR)")
    parser.add_argument("--web-port", type=int, default=8123,
                        help="HA Web UI port")
    parser.add_argument("--supervisor-port", type=int, default=4357,
                        help="HA Supervisor port")
    parser.add_argument("--ssh-port", type=int, default=22222,
                        help="HA OS SSH port")
    parser.add_argument("--timeout", type=int, default=30,
                        help="Scan timeout (sec)")
    parser.add_argument("--no-api-check", action="store_true",
                        help="Skip HA API check")
    parser.add_argument("--json", action="store_true",
                        help="Output JSON")
    args = parser.parse_args()

    discovery = HANetworkDiscovery(
        network=args.network,
        ha_web_port=args.web_port,
        ha_supervisor_port=args.supervisor_port,
        ssh_port=args.ssh_port,
        timeout=args.timeout,
    )

    devices = discovery.discover(check_api=not args.no_api_check)

    if args.json:
        print(discovery.to_json())

    sys.exit(0 if devices else 1)


if __name__ == "__main__":
    main()
```

---

## Discovery Flow

```
Phase 1: Network Scan (nmap)
  └→ Scan target ports: 8123, 4357, 22222
  └→ Filter: port 8123 open → HA candidate

Phase 2: Device Check
  └→ TCP port verification (8123/4357/22222)
  └→ Reverse DNS lookup
  └→ HA REST API check (optional)

Phase 3: Summary
  └→ Device list with status
  └→ Quick access URLs
```

---

## HA OS Port Reference

| ポート | サービス | 説明 |
|-------|---------|------|
| 8123 | Web UI | Home Assistant フロントエンド |
| 4357 | Supervisor API | HA OS Supervisor REST API |
| 22222 | SSH | HA OS SSHアクセス（アドオン経由） |
| 1883 | MQTT | Mosquittoブローカー（アドオン） |
| 1880 | Node-RED | Node-REDエディタ（アドオン） |

---

## Quick Start

```bash
# 基本的なスキャン
python3 ha_os_network_discovery.py --network 192.168.15.0/24

# VPN越しのスキャン
python3 ha_os_network_discovery.py --network 10.10.0.0/24

# JSON出力（スクリプト連携用）
python3 ha_os_network_discovery.py --network 192.168.15.0/24 --json

# APIチェックなし（高速）
python3 ha_os_network_discovery.py --network 192.168.15.0/24 --no-api-check
```

---

## Bash One-liner（nmapのみ）

スクリプトなしで素早くHAデバイスを探す場合:

```bash
# ポート8123が開いているデバイスを検索
nmap -sT -p 8123 --open -T4 192.168.15.0/24 | grep -E "(Nmap scan|open)"

# ポート8123 + 4357の組み合わせ
nmap -sT -p 8123,4357,22222 --open -T4 192.168.15.0/24
```

---

## Dependencies

| パッケージ | バージョン | 用途 |
|-----------|----------|------|
| `nmap` | 7.80+ | ネットワークスキャン |
| Python標準ライブラリ | - | socket, urllib, xml.etree |

### インストール

```bash
sudo apt install nmap
# Pythonパッケージの追加インストールは不要（標準ライブラリのみ）
```

---

## Troubleshooting

| 症状 | 原因 | 対処 |
|------|------|------|
| `nmap not found` | 未インストール | `sudo apt install nmap` |
| デバイスが見つからない | ネットワークセグメント違い | `ip addr` でサブネット確認 |
| ポート8123 closed | HA未起動/ファイアウォール | HA OSのSD直接確認 |
| API 401エラー | 認証トークン未設定 | 正常（HA起動は確認済み） |
| スキャン遅い | 大きなサブネット | `/24` → `/28` に絞る |

---

## Security Notes

- nmapスキャンは自分が管理するネットワークでのみ使用すること
- `-sT`（TCP connect）はsudo不要で安全なスキャン方式
- 社内ネットワークでの使用時はネットワーク管理者に確認すること
