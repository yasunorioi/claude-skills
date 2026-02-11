# IoT自動テスト環境ジェネレーター

> 統合済み: embedded-mqtt-tester, iot-mqtt-connection-tester の内容を含む

IoTデバイスの無人自動テスト環境を一括生成するスキル。
デバイス種類・トリガー方式・通知先・テスト項目を指定するだけで、
必要なスクリプトと設定ファイルをすべて生成する。

---

## 概要

殿が外出中でもIoTデバイスのテストを自動実行し、結果を通知するシステムを構築する。
USB接続検知、定時実行、手動実行に対応し、LINE/Slack/Discord/Emailで結果を通知できる。

### ユースケース

1. **開発時**: 新しいデバイスを接続したら自動でテストが走る
2. **外出時**: 定期テストの結果がスマホに届く
3. **デバッグ時**: 手動でテストを実行して問題を特定

---

## 使用方法

```
使用例:
「Pico 2 WのUSB接続テスト環境を作って。udevで自動検知、LINEで通知」
「W5500-EVB-PicoのLAN接続テスト。MQTT通信確認、cronで毎時実行」
「UniPiのリレー制御テスト環境を生成。Slackで通知」
```

### 入力パラメータ

| パラメータ | 必須 | 選択肢 | デフォルト |
|-----------|:----:|--------|-----------|
| デバイス種類 | ○ | usb, lan, wifi, unipi | - |
| デバイス名 | △ | 任意 | デバイス種類から推測 |
| トリガー方式 | - | udev, cron, manual | manual |
| 通知先 | - | line, slack, discord, email, none | none |
| テスト項目 | - | connection, mqtt, sensor, relay, all | connection |
| IPアドレス | △ | 任意（LAN/WiFi/UniPi時） | 192.168.1.100 |
| MQTTブローカー | △ | 任意（MQTT時） | localhost |

---

## 入力パラメータ詳細

### 1. デバイス種類 (device_type)

| 値 | 対象デバイス | 接続方式 |
|----|-------------|---------|
| `usb` | Pico 2 W, Pico等 | USB Serial (CircuitPython) |
| `lan` | W5500-EVB-Pico等 | Ethernet (固定IP/DHCP) |
| `wifi` | Pico 2 W等 | WiFi (DHCP) |
| `unipi` | UniPi 1.1 + EVOK | HTTP REST API |

### 2. トリガー方式 (trigger)

| 値 | 仕組み | 用途 |
|----|--------|------|
| `udev` | USB接続検知で自動起動 | USBデバイス向け |
| `cron` | systemd timerで定期実行 | LAN/WiFi/UniPi向け |
| `manual` | コマンドラインから手動実行 | デバッグ・初期確認 |

### 3. 通知先 (notification)

| 値 | サービス | 必要な設定 |
|----|---------|-----------|
| `line` | LINE Notify | トークン |
| `slack` | Slack Webhook | Webhook URL |
| `discord` | Discord Webhook | Webhook URL |
| `email` | SMTP送信 | SMTPサーバー設定 |
| `none` | 通知なし | - |

### 4. テスト項目 (test_items)

| 値 | 内容 | 対応デバイス |
|----|------|-------------|
| `connection` | 接続確認・Ping | 全て |
| `mqtt` | MQTT Pub/Sub | lan, wifi |
| `sensor` | センサー読み取り | usb, unipi |
| `relay` | リレー制御 | unipi |
| `all` | 全テスト項目 | デバイスに応じて |

---

## 出力ファイル構成

```
[output_dir]/
├── auto_test_runner.py      # メインテストスクリプト
├── test_config.json         # 設定ファイル
├── dummy_data_generator.py  # ダミーデータ生成（オプション）
├── requirements.txt         # Python依存パッケージ
├── systemd/                 # systemd関連（トリガーに応じて）
│   ├── iot-test.service     # テストサービス
│   └── iot-test.timer       # タイマー（cron時）
├── udev/                    # udev関連（USB時）
│   └── 99-iot-test.rules    # udevルール
├── notify/                  # 通知スクリプト
│   └── notify.py            # 通知共通モジュール
└── README.md                # セットアップ手順
```

---

## 生成されるコンポーネント

### 1. テストスクリプト (auto_test_runner.py)

```python
#!/usr/bin/env python3
"""
auto_test_runner.py - IoT自動テスト実行スクリプト

Usage:
    python3 auto_test_runner.py [--target TARGET] [--notify]
"""

import argparse
import json
import logging
import time
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, asdict
from typing import List

# ===== ログ設定 =====
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)


@dataclass
class TestResult:
    """テスト結果"""
    test_id: str
    name: str
    passed: bool
    duration_ms: int
    message: str
    timestamp: str


@dataclass
class TestReport:
    """テストレポート"""
    target: str
    total_tests: int
    passed: int
    failed: int
    duration_ms: int
    results: List[TestResult]
    timestamp: str


class TestRunner:
    """テスト実行基底クラス"""

    def __init__(self, config: dict):
        self.config = config
        self.results: List[TestResult] = []

    def run_test(self, test_id: str, name: str, test_func, timeout: float = 10.0) -> TestResult:
        """単一テストの実行"""
        start = time.time()
        try:
            result = test_func()
            passed = True
            message = str(result) if result else "OK"
        except Exception as e:
            passed = False
            message = f"Error: {str(e)}"

        duration = int((time.time() - start) * 1000)

        test_result = TestResult(
            test_id=test_id,
            name=name,
            passed=passed,
            duration_ms=duration,
            message=message,
            timestamp=datetime.now().isoformat()
        )

        self.results.append(test_result)
        logger.info(f"[{'PASS' if passed else 'FAIL'}] {test_id}: {name} ({duration}ms)")

        return test_result


# ===== デバイス固有のTestRunnerクラスがここに挿入される =====
# {{DEVICE_TEST_RUNNER}}


def main():
    parser = argparse.ArgumentParser(description="IoT Auto Test Runner")
    parser.add_argument("--target", default="all", help="Test target")
    parser.add_argument("--notify", action="store_true", help="Send notification")
    parser.add_argument("--config", default="test_config.json", help="Config file")
    args = parser.parse_args()

    # 設定読み込み
    config_path = Path(args.config)
    config = json.load(open(config_path)) if config_path.exists() else {}

    # テスト実行
    # {{TEST_EXECUTION}}

    # 通知送信
    if args.notify:
        # {{NOTIFICATION}}
        pass


if __name__ == "__main__":
    exit(main())
```

### 2. 設定ファイル (test_config.json)

```json
{
    "device_type": "{{DEVICE_TYPE}}",
    "device_name": "{{DEVICE_NAME}}",
    "target_ip": "{{TARGET_IP}}",
    "mqtt_broker": "{{MQTT_BROKER}}",
    "mqtt_port": 1883,
    "notification": {
        "type": "{{NOTIFICATION_TYPE}}",
        "token": "YOUR_TOKEN_HERE",
        "webhook_url": "YOUR_WEBHOOK_URL"
    },
    "test_timeout": 60,
    "relay_test_duration": 0.5
}
```

### 3. udevルール (99-iot-test.rules)

```udev
# IoTデバイス接続検知
# VID/PID はデバイスに応じて調整

ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="{{VENDOR_ID}}", \
    TAG+="systemd", ENV{SYSTEMD_WANTS}+="iot-test.service"

ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="{{VENDOR_ID}}", \
    RUN+="/usr/bin/logger -t iot-test 'Device connected: %k'"
```

### 4. systemdサービス (iot-test.service)

```ini
[Unit]
Description=IoT Device Test
After=network.target

[Service]
Type=oneshot
User={{USER}}
WorkingDirectory={{WORK_DIR}}
ExecStart=/usr/bin/python3 auto_test_runner.py --notify
StandardOutput=append:/var/log/iot-test/test.log
StandardError=append:/var/log/iot-test/test.log
```

### 5. systemdタイマー (iot-test.timer)

```ini
[Unit]
Description=IoT Periodic Test Timer

[Timer]
OnCalendar=*:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### 6. 通知モジュール (notify.py)

```python
#!/usr/bin/env python3
"""通知送信モジュール"""

import requests
import smtplib
from email.mime.text import MIMEText


def send_line_notify(token: str, message: str) -> bool:
    """LINE Notify送信"""
    headers = {"Authorization": f"Bearer {token}"}
    data = {"message": message}
    response = requests.post(
        "https://notify-api.line.me/api/notify",
        headers=headers,
        data=data
    )
    return response.status_code == 200


def send_slack_webhook(webhook_url: str, message: str) -> bool:
    """Slack Webhook送信"""
    payload = {"text": message}
    response = requests.post(webhook_url, json=payload)
    return response.status_code == 200


def send_discord_webhook(webhook_url: str, message: str) -> bool:
    """Discord Webhook送信"""
    payload = {"content": message}
    response = requests.post(webhook_url, json=payload)
    return response.status_code in [200, 204]


def send_email(smtp_config: dict, to_addr: str, subject: str, body: str) -> bool:
    """Email送信"""
    msg = MIMEText(body)
    msg["Subject"] = subject
    msg["From"] = smtp_config["from"]
    msg["To"] = to_addr

    with smtplib.SMTP(smtp_config["host"], smtp_config["port"]) as server:
        if smtp_config.get("use_tls"):
            server.starttls()
        if smtp_config.get("username"):
            server.login(smtp_config["username"], smtp_config["password"])
        server.send_message(msg)
    return True
```

---

## サンプル出力：Pico 2 W USB接続テスト

### 入力

```
デバイス種類: usb
デバイス名: Pico 2 W
トリガー: udev
通知先: line
テスト項目: all
```

### 生成ファイル

#### auto_test_runner.py (USB版)

```python
#!/usr/bin/env python3
"""
Pico 2 W USB自動テストスクリプト
"""

import serial
import serial.tools.list_ports
import time
import json
import logging
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, asdict
from typing import List, Optional

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')
logger = logging.getLogger(__name__)


@dataclass
class TestResult:
    test_id: str
    name: str
    passed: bool
    duration_ms: int
    message: str
    timestamp: str


class USBTestRunner:
    """Pico 2 W USB接続テスト"""

    PICO_VID = "2E8A"  # Raspberry Pi Foundation

    def __init__(self, config: dict):
        self.config = config
        self.results: List[TestResult] = []
        self.port = None
        self.serial = None

    def run_test(self, test_id: str, name: str, test_func, timeout: float = 10.0) -> TestResult:
        start = time.time()
        try:
            result = test_func()
            passed = True
            message = str(result) if result else "OK"
        except Exception as e:
            passed = False
            message = f"Error: {str(e)}"

        duration = int((time.time() - start) * 1000)
        test_result = TestResult(test_id, name, passed, duration, message, datetime.now().isoformat())
        self.results.append(test_result)
        logger.info(f"[{'PASS' if passed else 'FAIL'}] {test_id}: {name} ({duration}ms)")
        return test_result

    def find_pico_port(self) -> Optional[str]:
        """Picoのシリアルポートを検索"""
        for port in serial.tools.list_ports.comports():
            if port.vid and f"{port.vid:04X}" == self.PICO_VID:
                return port.device
        return None

    def test_device_recognition(self) -> str:
        """USB-001: デバイス認識"""
        self.port = self.find_pico_port()
        if not self.port:
            raise Exception("Pico device not found")
        return f"Found at {self.port}"

    def test_serial_connection(self) -> str:
        """USB-002: シリアル接続"""
        if not self.port:
            raise Exception("No port available")
        self.serial = serial.Serial(self.port, 115200, timeout=3)
        return "Connected at 115200bps"

    def test_repl_response(self) -> str:
        """USB-003: REPL応答"""
        if not self.serial:
            raise Exception("Serial not connected")
        self.serial.write(b'\x03')  # Ctrl+C
        time.sleep(0.5)
        self.serial.write(b'\r\n')
        response = self.serial.read(100).decode('utf-8', errors='ignore')
        if '>>>' in response:
            return "REPL active"
        raise Exception(f"No REPL prompt: {response[:50]}")

    def test_cpu_temperature(self) -> str:
        """USB-004: CPU温度読み取り"""
        if not self.serial:
            raise Exception("Serial not connected")
        cmd = "import microcontroller; print(microcontroller.cpu.temperature)\r\n"
        self.serial.write(cmd.encode())
        time.sleep(1)
        response = self.serial.read(200).decode('utf-8', errors='ignore')
        for line in response.split('\n'):
            try:
                temp = float(line.strip())
                if 20 <= temp <= 80:
                    return f"{temp}C"
            except ValueError:
                continue
        raise Exception(f"Invalid temperature: {response[:50]}")

    def test_echo(self) -> str:
        """USB-005: エコーテスト"""
        if not self.serial:
            raise Exception("Serial not connected")
        test_value = f"ECHO_{int(time.time() * 1000) % 100000}"
        cmd = f"print('{test_value}')\r\n"
        self.serial.write(cmd.encode())
        time.sleep(0.5)
        response = self.serial.read(200).decode('utf-8', errors='ignore')
        if test_value in response:
            return f"Echo OK: {test_value}"
        raise Exception(f"Echo failed: expected {test_value}")

    def run_all(self) -> dict:
        """全テスト実行"""
        start = time.time()

        self.run_test("USB-001", "デバイス認識", self.test_device_recognition, 10)
        self.run_test("USB-002", "シリアル接続", self.test_serial_connection, 5)
        self.run_test("USB-003", "REPL応答", self.test_repl_response, 3)
        self.run_test("USB-004", "CPU温度読み取り", self.test_cpu_temperature, 5)
        self.run_test("USB-005", "エコーテスト", self.test_echo, 5)

        if self.serial:
            self.serial.close()

        passed = sum(1 for r in self.results if r.passed)
        return {
            "target": "USB (Pico 2 W)",
            "total_tests": len(self.results),
            "passed": passed,
            "failed": len(self.results) - passed,
            "duration_ms": int((time.time() - start) * 1000),
            "results": [asdict(r) for r in self.results],
            "timestamp": datetime.now().isoformat()
        }


def send_line_notify(token: str, message: str) -> bool:
    import requests
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.post("https://notify-api.line.me/api/notify", headers=headers, data={"message": message})
    return response.status_code == 200


def main():
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--notify", action="store_true")
    parser.add_argument("--config", default="test_config.json")
    args = parser.parse_args()

    config = json.load(open(args.config)) if Path(args.config).exists() else {}

    runner = USBTestRunner(config)
    report = runner.run_all()

    # 結果表示
    print("\n" + "=" * 50)
    print(f"TEST SUMMARY: {report['target']}")
    print("=" * 50)
    print(f"Passed: {report['passed']}/{report['total_tests']}")
    for r in report['results']:
        status = "PASS" if r['passed'] else "FAIL"
        print(f"  [{status}] {r['test_id']}: {r['name']}")

    # 結果保存
    output = Path(f"test_result_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json")
    json.dump(report, open(output, "w"), indent=2)
    logger.info(f"Results saved to {output}")

    # LINE通知
    if args.notify and config.get("notification", {}).get("token"):
        message = f"\n【Pico 2 W テスト結果】\n"
        message += f"結果: {report['passed']}/{report['total_tests']} 成功\n"
        for r in report['results']:
            status = "OK" if r['passed'] else "NG"
            message += f"{status} {r['name']}\n"

        if send_line_notify(config["notification"]["token"], message):
            logger.info("LINE notification sent")

    return 0 if report['failed'] == 0 else 1


if __name__ == "__main__":
    exit(main())
```

#### test_config.json

```json
{
    "device_type": "usb",
    "device_name": "Pico 2 W",
    "notification": {
        "type": "line",
        "token": "YOUR_LINE_NOTIFY_TOKEN"
    },
    "test_timeout": 60
}
```

#### udev/99-iot-test.rules

```udev
# Pico 2 W 接続検知
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="2e8a", \
    TAG+="systemd", ENV{SYSTEMD_WANTS}+="iot-test.service"

ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="2e8a", \
    RUN+="/usr/bin/logger -t iot-test 'Pico 2 W connected: %k'"
```

#### systemd/iot-test.service

```ini
[Unit]
Description=Pico 2 W Auto Test
After=network.target

[Service]
Type=oneshot
User=yasu
WorkingDirectory=/home/yasu/iot-test
ExecStart=/usr/bin/python3 auto_test_runner.py --notify
StandardOutput=append:/var/log/iot-test/test.log
StandardError=append:/var/log/iot-test/test.log
```

#### requirements.txt

```
pyserial>=3.5
requests>=2.28.0
```

#### README.md

```markdown
# Pico 2 W USB自動テスト環境

## セットアップ

1. 依存パッケージのインストール
pip3 install -r requirements.txt

2. ログディレクトリ作成
sudo mkdir -p /var/log/iot-test
sudo chown $USER:$USER /var/log/iot-test

3. udevルール配置
sudo cp udev/99-iot-test.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules

4. systemdサービス配置
sudo cp systemd/iot-test.service /etc/systemd/system/
sudo systemctl daemon-reload

5. LINE Notifyトークン設定
test_config.json の notification.token を設定

## 使用方法

### 手動実行
python3 auto_test_runner.py --notify

### 自動実行
Pico 2 Wを接続すると自動でテストが実行される

## ログ確認
tail -f /var/log/iot-test/test.log
```

---

## テストケース一覧

### USB接続テスト

| ID | テスト項目 | 期待結果 | タイムアウト |
|----|-----------|---------|-------------|
| USB-001 | デバイス認識 | /dev/ttyACM* が出現 | 10s |
| USB-002 | シリアル接続 | 115200bps で接続成功 | 5s |
| USB-003 | REPL応答 | プロンプト表示 | 3s |
| USB-004 | CPU温度読み取り | 20-80度の範囲内 | 5s |
| USB-005 | エコーテスト | 送信データ返却 | 5s |

### LAN接続テスト

| ID | テスト項目 | 期待結果 | タイムアウト |
|----|-----------|---------|-------------|
| LAN-001 | リンク検出 | arp-scan でMAC検出 | 30s |
| LAN-002 | Ping応答 | ICMPエコー成功 | 5s |
| LAN-003 | MQTT接続 | broker接続成功 | 10s |
| LAN-004 | MQTT Publish | メッセージ送信成功 | 5s |
| LAN-005 | MQTT Subscribe | メッセージ受信成功 | 10s |

### UniPiテスト

| ID | テスト項目 | 期待結果 | タイムアウト |
|----|-----------|---------|-------------|
| UNI-001 | API応答 | /rest/all が200 OK | 5s |
| UNI-002 | リレー状態取得 | 成功 | 3s |
| UNI-003 | リレーON | 成功 | 3s |
| UNI-004 | リレーOFF | 成功 | 3s |

---

## 安全ガイドライン

### リレーテスト時の注意

1. **負荷なし状態でテスト**: 機器が接続されていない状態で実行
2. **短時間ON**: ON後0.5秒以内にOFF
3. **タイムアウト時は強制OFF**: 異常時の安全確保

```python
# 安全なリレーテストパターン
try:
    relay_on()
    time.sleep(0.5)
finally:
    relay_off()  # 必ずOFF
```

### タイムアウト設定

| 操作 | 推奨タイムアウト |
|------|---------------|
| USB認識 | 10秒 |
| シリアル通信 | 5秒 |
| MQTT接続 | 10秒 |
| リレー操作 | 3秒 |
| テスト全体 | 5分 |

---

## 関連スキル

- `unipi-arsprout-integration`: UniPi EVOK API詳細リファレンス
- `uecs-mqtt-bridge-generator`: UECS CCM-MQTT変換ブリッジ生成
- `iot-system-spec-generator`: IoTシステム仕様書テンプレート

---

## 参考資料

- `/home/yasu/arsprout_analysis/docs/AUTO_TEST_DESIGN.md` - 元設計書
- LINE Notify: https://notify-bot.line.me/ja/
- Slack Webhook: https://api.slack.com/messaging/webhooks
- Discord Webhook: https://discord.com/developers/docs/resources/webhook

---

## 統合元からの補足

### embedded-mqtt-tester からの補足 (CircuitPython MQTT テスト)

CircuitPython/MicroPython デバイス向けのREPL経由MQTTテストパターン（参考情報）:

**REPL制御クラス (`REPLController`)**:
- `_find_pico_port()`: Pico VID=2E8Aで自動検出
- `execute(command, timeout)`: REPLコマンド送信・応答取得（`>>>`プロンプト検知）
- `soft_reboot()`: Ctrl+D（0x04）でソフトリブート

**デバイス用テストフィクスチャ (`mqtt_test_fixture.py`)**: デバイス側に配置する`/lib/mqtt_test_fixture.py`。CircuitPython 10.x対応:
- SocketPool API (`adafruit_wiznet5k_socketpool`)を使用
- W5500/WiFi自動検出（`NETWORK_TYPE`）
- `test_connect()`, `test_publish()`, `test_subscribe_async()` メソッド

**CircuitPython 10.x APIマイグレーション**:
- 9.x: `MQTT.set_socket(socket, eth)` → 10.x: `MQTT(..., socket_pool=SocketPool(eth))`
- バージョン検出: `sys.implementation.version[:2]`

**mosquitto背景購読パターン**: `mosquitto_sub`を`subprocess.Popen()`でバックグラウンド実行し、デバイスPublishを検証。

### iot-mqtt-connection-tester からの補足 (paho-mqtt テスト)

paho-mqttベースの汎用MQTT接続テストパターン（Linux/Python 3.10+向け）:

**テストシナリオ**:
1. **TCP接続**: `socket.create_connection()` でTCP層疎通確認
2. **MQTT接続**: `client.connect()` + CONNACKのRC確認（RC=0なら成功）
3. **Publish検証**: `publish().wait_for_publish()` で送信完了確認
4. **Subscribe検証**: テストメッセージ送信→`on_message`コールバックで受信確認
5. **往復遅延計測**: Pub→Sub遅延を複数回計測（平均1000ms未満を合格基準）
6. **LWT設定テスト**: `will_set()`で遺言設定→接続成功を確認

**paho-mqtt 2.x APIパターン**:
- `callback_api_version=mqtt.CallbackAPIVersion.VERSION2` 必須
- `on_connect(client, userdata, flags, reason_code, properties)` シグネチャ
- `reason_code.value` でRC取得

**結果構造化出力**:
- JSON形式（`--json`オプション）でCI/CD統合対応
- 各テストに`passed`, `detail`, `timestamp`を記録

**認証・TLS対応**:
- `client.username_pw_set(username, password)` で認証
- `client.tls_set()` でTLS有効化（ポート8883推奨）

**RCコード文字列変換**: `_rc_to_string(rc)` で0=Connection Accepted, 4=Bad Username/Password, 5=Not Authorized等の判定。

**テストマトリクス参考**:
| テスト | 確認内容 | 合格条件 |
|--------|---------|---------|
| TCP接続 | ブローカーへのTCPリーチ | ポート到達可能 |
| MQTT接続 | CONNECT/CONNACK | RC=0 |
| Publish | メッセージ送信 | is_published()=True |
| Subscribe | メッセージ受信 | タイムアウト内に1件以上受信 |
| 往復遅延 | Pub→Sub遅延 | 平均1000ms未満 |
| LWT設定 | Last Will設定 | 接続成功 |
