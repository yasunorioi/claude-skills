# w5500-evb-pico-guide

> 統合済み: w5500-ethernet-connection-test の内容を含む

W5500-EVB-Pico / Pico2 の使い方ガイド（PoEイーサネットボード）

## 概要

WIZnet製のRP2040/RP2350搭載イーサネット開発ボードの使い方をガイドするスキル。
PoE給電対応で、電源ケーブル不要のIoTノード構築に最適。

## 使用方法

```
/w5500-evb-pico-guide [トピック]
```

### トピック例

```
/w5500-evb-pico-guide overview     # ボード概要
/w5500-evb-pico-guide poe          # PoE給電仕様
/w5500-evb-pico-guide pinout       # ピン配置
/w5500-evb-pico-guide setup        # CircuitPython初期化手順
/w5500-evb-pico-guide mqtt         # MQTT接続サンプル
/w5500-evb-pico-guide troubleshoot # トラブルシューティング
```

## 1. ボード概要

### 製品情報

| 項目 | W5500-EVB-Pico | W5500-EVB-Pico2 |
|------|----------------|-----------------|
| メーカー | WIZnet | WIZnet |
| MCU | RP2040 | RP2350 |
| イーサネット | W5500 (10/100Mbps) | W5500 (10/100Mbps) |
| 購入先 | 秋月電子 ¥1,650 / DigiKey | 秋月電子 ¥2,420 / DigiKey |
| PoE対応版 | W5500-EVB-Pico-PoE | W5500-EVB-Pico2-PoE |

### W5500チップの特徴

- **ハードワイヤードTCP/IP**: ソフトウェアスタック不要で軽量
- **8ソケット同時接続**: 複数接続を並行処理
- **32KB内蔵バッファ**: 送受信各16KB
- **SPI接続**: 最大80MHz
- **対応プロトコル**: TCP, UDP, IPv4, ICMP, ARP, IGMP, PPPoE
- **WOL対応**: Wake on LAN機能

## 2. PoE給電の仕様

### PoE対応モデル（-PoE付き）

| 項目 | 仕様 |
|------|------|
| PoEモジュール | WIZPoE-P1 |
| 対応規格 | IEEE 802.3af (Mode A/B両対応) |
| 受電電力 | 最大7W |
| 出力電圧 | 5V (VSYS供給) |

### PoEスイッチ/インジェクター要件

- **IEEE 802.3af準拠**のPoEスイッチまたはインジェクター
- Class 0～3対応（7W以上供給可能）

### USBとPoEの同時使用

| 状況 | 動作 |
|------|------|
| PoEのみ | PoEから給電（5V出力） |
| USBのみ | USBから給電（5V VBUS） |
| 両方接続 | **USB優先**（PoEからの給電は自動カット） |

**注意**: 開発中はUSB接続でシリアルコンソールを使用。運用時はPoEのみで動作。

## 3. ピン配置

### W5500専用ピン（使用不可）

```
┌─────────────────────────────────────┐
│  W5500 SPI Interface (Reserved)    │
├──────────┬──────────┬──────────────┤
│ GP16     │ MISO     │ SPI RX       │
│ GP17     │ CS       │ Chip Select  │
│ GP18     │ SCK      │ SPI Clock    │
│ GP19     │ MOSI     │ SPI TX       │
│ GP20     │ RST      │ Reset        │
│ GP21     │ INT      │ Interrupt    │
└──────────┴──────────┴──────────────┘
```

**重要**: GP16-21はW5500が使用するため、他の用途に使えない。

### 汎用GPIO（使用可能）

```
┌─────────────────────────────────────┐
│  Available GPIO Pins               │
├──────────┬──────────┬──────────────┤
│ GP0-GP15 │ 汎用     │ 自由に使用可 │
│ GP22     │ 汎用     │ 自由に使用可 │
│ GP26     │ ADC0     │ アナログ入力 │
│ GP27     │ ADC1     │ アナログ入力 │
│ GP28     │ ADC2     │ アナログ入力 │
└──────────┴──────────┴──────────────┘
```

### I2C（STEMMA QT/Qwiic互換）

```
┌─────────────────────────────────────┐
│  I2C (STEMMA_I2C)                  │
├──────────┬──────────┬──────────────┤
│ GP4      │ SDA      │ I2C Data     │
│ GP5      │ SCL      │ I2C Clock    │
└──────────┴──────────┴──────────────┘
```

### 特殊ピン

| ピン | 機能 | 備考 |
|------|------|------|
| GP24 | VBUS_SENSE | USB接続検出 |
| GP25 | LED | ユーザーLED |
| GP29 | VSYS/3 | 電源電圧モニター（ADC） |

### ボード図（簡易）

```
        ┌────────────────────────────┐
        │    W5500-EVB-Pico(-PoE)    │
        │                            │
USB ────┤ [BOOTSEL] [LED]           ├──── RJ45
        │                            │    Ethernet
        │  GP0  ●────────────● GP15 │
        │  GP1  ●────────────● GP14 │
        │  GP2  ●────────────● GP13 │
        │  GP3  ●────────────● GP12 │
        │  GP4  ● SDA    SCL ● GP5  │ ← I2C
        │  GP6  ●────────────● GP11 │
        │  GP7  ●────────────● GP10 │
        │  GP8  ●────────────● GP9  │
        │                            │
        │  GP26 ● ADC0   ADC1 ● GP27│
        │  GP28 ● ADC2   LED  ● GP25│
        │                            │
        │  [W5500: GP16-21 Reserved] │
        └────────────────────────────┘
```

## 4. CircuitPython初期化手順

### Step 1: UF2ファイルのダウンロード

| ボード | ダウンロードURL |
|--------|----------------|
| W5500-EVB-Pico | https://circuitpython.org/board/wiznet_w5500_evb_pico/ |
| W5500-EVB-Pico2 | https://circuitpython.org/board/wiznet_w5500_evb_pico2/ |

### Step 2: BOOTSELモードで書き込み

1. BOOTSELボタンを押しながらUSB接続
2. 「RPI-RP2」ドライブが表示される
3. .uf2ファイルをドラッグ&ドロップ
4. 自動再起動後「CIRCUITPY」ドライブが表示

### Step 3: ライブラリインストール

**circupを使用（推奨）**:
```bash
pip install circup
circup install adafruit_wiznet5k adafruit_minimqtt adafruit_requests
```

**手動インストール**:
1. CircuitPython Library Bundleをダウンロード
2. 以下をCIRCUITPY/lib/にコピー:
   - `adafruit_wiznet5k/` (フォルダ)
   - `adafruit_minimqtt/` (フォルダ)
   - `adafruit_requests.mpy`
   - `adafruit_connection_manager.mpy`
   - `adafruit_ticks.mpy`

### Step 4: 初期化コード

```python
import board
import busio
import digitalio
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K

# SPI設定（W5500-EVB-Pico専用ピン）
spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)
cs = digitalio.DigitalInOut(board.GP17)
rst = digitalio.DigitalInOut(board.GP20)

# W5500初期化（DHCP有効）
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)

# 接続確認
print("MAC:", [hex(i) for i in eth.mac_address])
print("IP:", eth.pretty_ip(eth.ip_address))
```

## 5. adafruit_wiznet5k ライブラリ使用法

### WIZNET5K クラス

```python
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K

# 基本初期化（DHCP）
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)

# 静的IP設定
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=False)
eth.ifconfig = (
    (192, 168, 1, 100),  # IP
    (255, 255, 255, 0),  # Subnet
    (192, 168, 1, 1),    # Gateway
    (8, 8, 8, 8)         # DNS
)
```

### MACアドレス設定

```python
# デフォルトMAC: de:ad:be:ef:fe:ed
# カスタムMAC設定
eth = WIZNET5K(spi, cs, reset=rst,
               mac=(0x00, 0x11, 0x22, 0x33, 0x44, 0x55),
               is_dhcp=True)
```

### ネットワーク情報取得

```python
print("MAC:", ":".join([hex(i)[2:] for i in eth.mac_address]))
print("IP:", eth.pretty_ip(eth.ip_address))
print("Link:", "Up" if eth.link_status else "Down")
```

## 6. DHCP/固定IP設定

### DHCP自動取得

```python
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)
# 自動的にIP、サブネット、ゲートウェイ、DNSを取得
```

### 静的IP設定

```python
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=False)
eth.ifconfig = (
    (192, 168, 15, 100),   # IP Address
    (255, 255, 255, 0),    # Subnet Mask
    (192, 168, 15, 1),     # Gateway
    (192, 168, 15, 1)      # DNS Server
)
```

### settings.toml からの設定読み込み

```python
import os

use_dhcp = os.getenv("USE_DHCP", "true").lower() == "true"

if use_dhcp:
    eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)
else:
    eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=False)
    eth.ifconfig = (
        tuple(map(int, os.getenv("STATIC_IP", "192.168.1.100").split("."))),
        tuple(map(int, os.getenv("SUBNET", "255.255.255.0").split("."))),
        tuple(map(int, os.getenv("GATEWAY", "192.168.1.1").split("."))),
        tuple(map(int, os.getenv("DNS", "8.8.8.8").split(".")))
    )
```

## 7. MQTT接続サンプルコード

### CircuitPython 10.x 対応（SocketPool API）

```python
import board
import busio
import digitalio
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool
import adafruit_minimqtt.adafruit_minimqtt as MQTT

# W5500初期化
spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)
cs = digitalio.DigitalInOut(board.GP17)
rst = digitalio.DigitalInOut(board.GP20)
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)

print("IP:", eth.pretty_ip(eth.ip_address))

# SocketPool作成（CircuitPython 10.x必須）
pool = SocketPool(eth)

# MQTTクライアント作成
mqtt = MQTT.MQTT(
    broker="192.168.15.14",
    port=1883,
    socket_pool=pool,
    client_id="w5500-pico-01"
)

# コールバック設定
def on_connect(client, userdata, flags, rc):
    print("Connected!")

def on_message(client, topic, message):
    print(f"Message: {topic} = {message}")

mqtt.on_connect = on_connect
mqtt.on_message = on_message

# 接続・送受信
mqtt.connect()
mqtt.subscribe("sensor/#")
mqtt.publish("status/w5500", "online")

# メインループ
while True:
    mqtt.loop()
```

### 完全なセンサー送信例

```python
import board
import busio
import digitalio
import time
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool
import adafruit_minimqtt.adafruit_minimqtt as MQTT
import adafruit_sht4x  # SHT40センサー

# ハードウェア初期化
spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)
cs = digitalio.DigitalInOut(board.GP17)
rst = digitalio.DigitalInOut(board.GP20)
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)

i2c = board.STEMMA_I2C()
sht = adafruit_sht4x.SHT4x(i2c)

# MQTT設定
pool = SocketPool(eth)
mqtt = MQTT.MQTT(broker="192.168.15.14", port=1883, socket_pool=pool)
mqtt.connect()

# センサー値送信ループ
while True:
    temp, humidity = sht.measurements
    mqtt.publish("sensor/temperature", f"{temp:.1f}")
    mqtt.publish("sensor/humidity", f"{humidity:.1f}")
    time.sleep(30)
```

## 8. CircuitPython 10.x API変更点

### 旧API（CircuitPython 9.x以前）

```python
# 旧方式（廃止）
import adafruit_wiznet5k.adafruit_wiznet5k_socket as socket
MQTT.set_socket(socket, eth)
mqtt = MQTT.MQTT(broker="...", port=1883)
```

### 新API（CircuitPython 10.x）

```python
# 新方式（必須）
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool
pool = SocketPool(eth)
mqtt = MQTT.MQTT(broker="...", port=1883, socket_pool=pool)
```

### 主な変更点

| 項目 | 旧API (9.x) | 新API (10.x) |
|------|-------------|--------------|
| モジュール | `adafruit_wiznet5k_socket` | `adafruit_wiznet5k_socketpool` |
| 初期化 | `MQTT.set_socket(socket, eth)` | `socket_pool=pool` パラメータ |
| SocketPool | 不要 | `SocketPool(eth)` 必須 |

### 移行チェックリスト

- [ ] `import` 文を `_socketpool` に変更
- [ ] `SocketPool(eth)` でプール作成
- [ ] `MQTT.MQTT()` に `socket_pool=pool` を追加
- [ ] `MQTT.set_socket()` 呼び出しを削除

## 9. トラブルシューティング

### Link LEDが点灯しない

| 原因 | 対処法 |
|------|--------|
| LANケーブル未接続 | ケーブルを確認、両端のコネクタをしっかり挿す |
| ケーブル不良 | 別のケーブルを試す |
| スイッチ/ルーター問題 | 他のポートを試す |
| W5500未初期化 | コードでW5500初期化を確認 |

### DHCPでIPが取得できない

```python
# デバッグ用コード
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True, debug=True)
print("Link:", eth.link_status)
print("IP:", eth.pretty_ip(eth.ip_address))
```

| 原因 | 対処法 |
|------|--------|
| DHCPサーバーなし | 静的IP設定に変更 |
| ネットワーク分離 | VLANやファイアウォール確認 |
| 初期化タイミング | `time.sleep(2)` を追加して待機 |

### MQTTブローカーに接続できない

```
ConnectionError: Failed to establish connection.
MMQTTException: ('Repeated connect failures', None)
```

| 原因 | 対処法 |
|------|--------|
| ブローカーIP間違い | IPアドレス確認、ping可能か確認 |
| ポート間違い | 通常1883（非TLS）、8883（TLS） |
| ブローカー未起動 | `mosquitto_sub -t "#"` で確認 |
| ファイアウォール | ポート1883を開放 |

### ImportError: ライブラリ不足

```
ImportError: no module named 'adafruit_wiznet5k'
```

| 対処法 |
|--------|
| circupでインストール: `circup install adafruit_wiznet5k` |
| 手動: Library Bundleから `adafruit_wiznet5k/` をコピー |
| バージョン確認: CircuitPythonとBundleのメジャーバージョンを合わせる |

### メモリ不足（MemoryError）

```python
# 軽量化のヒント
import gc
gc.collect()
print("Free memory:", gc.mem_free())
```

| 対処法 |
|--------|
| `.mpy` ファイルを使用（.pyより小さい） |
| 不要なライブラリを削除 |
| 変数を使い終わったら `del` で解放 |

## 10. Grove Shield for Pi Pico とのスタック構成

### 概要

W5500-EVB-Pico-PoE と Grove Shield for Pi Pico を物理的にスタック（積み重ね）することで、
**PoE給電 + 有線LAN + Grove接続** の便利な構成を実現できる。

### スタック構成図

```
┌────────────────────────────────┐
│  Grove Shield for Pi Pico     │ ← 上段
│  ┌─────┐ ┌─────┐ ┌─────┐     │
│  │I2C-1│ │I2C-2│ │UART │ ... │    Grove端子群
│  └─────┘ └─────┘ └─────┘     │
├────────────────────────────────┤
│  W5500-EVB-Pico-PoE           │ ← 下段
│  [USB] [BOOTSEL] [RJ45]       │
└────────────────────────────────┘
       ↑PoE給電 + Ethernet
```

### Grove Shield for Pi Pico 仕様

| 項目 | 仕様 |
|------|------|
| メーカー | Seeed Studio |
| Grove端子 | I2C×2, UART×2, Analog×3, Digital×3 |
| SPI端子 | 1ポート |
| SWD端子 | デバッグ用 |
| 電圧切替 | 3.3V / 5V スイッチ |
| サイズ | 56mm × 56mm |

### ピン干渉の確認

**W5500が使用するピン（Grove Shieldで使用不可）**:

| ピン | W5500機能 | Grove Shield |
|------|-----------|--------------|
| GP16-21 | SPI (W5500) | **使用不可** |

**Grove Shieldで使用可能なピン**:

| Grove端子 | ピン | 用途 | 干渉 |
|-----------|------|------|------|
| I2C-1 | GP4/GP5 | I2Cセンサー | なし |
| I2C-2 | GP6/GP7 | I2Cセンサー | なし |
| UART-1 | GP0/GP1 | シリアル通信 | なし |
| UART-2 | GP8/GP9 | シリアル通信 | なし |
| Analog A0 | GP26 | アナログセンサー | なし |
| Analog A1 | GP27 | アナログセンサー | なし |
| Analog A2 | GP28 | アナログセンサー | なし |
| Digital D16 | GP16 | デジタルI/O | **W5500使用中** |

**結論**: GP16-21以外のGrove端子は問題なく使用可能。

### スタック時の考慮事項

| 項目 | 注意点 |
|------|--------|
| 高さ | スタック時 約15mm増加 |
| RJ45アクセス | 下段なので問題なし |
| USBアクセス | 下段側面から接続可能 |
| 電圧設定 | Grove Shieldの3.3V/5Vスイッチ確認 |
| ケース | 3Dプリント等でカスタムケース推奨 |

### I2C使用時のコード例

```python
import board
import busio

# Grove Shield I2C-1 (GP4/GP5) - STEMMA_I2Cと同じ
i2c1 = board.STEMMA_I2C()

# Grove Shield I2C-2 (GP6/GP7) - 別のI2Cバス
i2c2 = busio.I2C(board.GP7, board.GP6)

# I2Cデバイススキャン
while not i2c1.try_lock():
    pass
print("I2C-1 devices:", [hex(x) for x in i2c1.scan()])
i2c1.unlock()
```

## 11. M5Stackセンサー（Grove端子）との接続例

### M5Stack Unit シリーズの利点

- **ケース付き**: はんだ付け不要、すぐに使える
- **Grove端子**: 4ピンコネクタで簡単接続
- **多彩なラインナップ**: 温湿度、CO2、照度、距離など
- **I2C通信**: 複数センサーをデイジーチェーン可能

### 対応センサー一覧

| Unit名 | センサー | I2Cアドレス | 測定項目 |
|--------|----------|-------------|----------|
| ENV III | SHT30 + QMP6988 | 0x44, 0x70 | 温度、湿度、気圧 |
| TVOC/eCO2 | SGP30 | 0x58 | TVOC、eCO2 |
| DLight | BH1750 | 0x23 | 照度 (lux) |
| PIR | AS312 | - (Digital) | 人感検知 |
| ToF | VL53L0X | 0x29 | 距離 (mm) |

### Unit ENV III（温湿度・気圧）

**仕様**:
- SHT30: 温度 -40～125°C、湿度 0～100%RH
- QMP6988: 気圧 300～1100hPa

**CircuitPythonコード**:
```python
import board
import adafruit_sht31d  # SHT30互換
# QMP6988用ライブラリは別途必要

i2c = board.STEMMA_I2C()

# SHT30 (アドレス 0x44)
sht = adafruit_sht31d.SHT31D(i2c)
temp = sht.temperature
humidity = sht.relative_humidity
print(f"Temp: {temp:.1f}°C, Humidity: {humidity:.1f}%")
```

**必要ライブラリ**:
```bash
circup install adafruit_sht31d
```

### Unit TVOC/eCO2（空気品質）

**仕様**:
- SGP30: TVOC 0～60000 ppb、eCO2 400～60000 ppm

**CircuitPythonコード**:
```python
import board
import adafruit_sgp30

i2c = board.STEMMA_I2C()

# SGP30 (アドレス 0x58)
sgp30 = adafruit_sgp30.Adafruit_SGP30(i2c)
sgp30.iaq_init()

# 測定（15秒ウォームアップ推奨）
eco2, tvoc = sgp30.iaq_measure()
print(f"eCO2: {eco2} ppm, TVOC: {tvoc} ppb")
```

**必要ライブラリ**:
```bash
circup install adafruit_sgp30
```

### Unit DLight（照度）

**仕様**:
- BH1750: 1～65535 lux

**CircuitPythonコード**:
```python
import board
import adafruit_bh1750

i2c = board.STEMMA_I2C()

# BH1750 (アドレス 0x23)
light = adafruit_bh1750.BH1750(i2c)
lux = light.lux
print(f"Light: {lux:.1f} lux")
```

**必要ライブラリ**:
```bash
circup install adafruit_bh1750
```

### 複数センサー統合例

```python
import board
import time
import adafruit_sht31d
import adafruit_sgp30
import adafruit_bh1750

i2c = board.STEMMA_I2C()

# センサー初期化
sht = adafruit_sht31d.SHT31D(i2c)
sgp = adafruit_sgp30.Adafruit_SGP30(i2c)
light = adafruit_bh1750.BH1750(i2c)

sgp.iaq_init()

while True:
    # 全センサー読み取り
    temp = sht.temperature
    humidity = sht.relative_humidity
    eco2, tvoc = sgp.iaq_measure()
    lux = light.lux

    print(f"Temp:{temp:.1f}C Hum:{humidity:.0f}% CO2:{eco2}ppm TVOC:{tvoc}ppb Lux:{lux:.0f}")
    time.sleep(5)
```

### 接続図

```
┌─────────────────────────────────────────────────┐
│            Grove Shield for Pi Pico             │
│  ┌─────┐    ┌─────┐    ┌─────┐                 │
│  │I2C-1│    │I2C-2│    │ ... │                 │
│  └──┬──┘    └──┬──┘    └─────┘                 │
└─────┼──────────┼────────────────────────────────┘
      │          │
      │    ┌─────┴─────┐
      │    │ Grove Hub │ (オプション)
      │    └─┬───┬───┬─┘
      │      │   │   │
   ┌──┴──┐ ┌─┴─┐ ┌┴──┐ ┌───┐
   │ENV  │ │SGP│ │BH │ │...│
   │III  │ │30 │ │175│ │   │
   └─────┘ └───┘ └───┘ └───┘
```

### Grove Hub の活用

複数のI2Cセンサーを接続する場合、Grove I2C Hubを使用:
- **Grove - I2C Hub (6 Port)**: 1つのI2Cポートを6分岐
- アドレスが異なるセンサーなら同一バスで動作

## 12. 実テスト結果（2026-02-05実施）

### テスト環境

| 項目 | 値 |
|------|---|
| ボード | W5500-EVB-Pico2 |
| CircuitPython | 10.0.3 |
| MQTTブローカー | Mosquitto (Docker: arsprout-mqtt) |
| ネットワーク | 192.168.15.0/24 |

### テスト結果サマリ

| テスト項目 | 結果 | 詳細 |
|-----------|------|------|
| USB CDC（シリアル通信） | ✅ OK | /dev/ttyACM0、115200bps |
| CircuitPython REPL | ✅ OK | Ctrl+C, Ctrl+D 正常動作 |
| GPIO制御（LED） | ✅ OK | board.LED 点灯/消灯 |
| W5500初期化 | ✅ OK | SPI経由で認識 |
| MACアドレス取得 | ✅ OK | de:ad:be:ef:fe:ed |
| DHCP IP取得 | ✅ OK | 192.168.15.13 |
| SocketPool作成 | ✅ OK | CircuitPython 10.x API |
| MQTT接続（Mosquitto） | ✅ OK | 192.168.15.14:1883 |
| MQTT Publish | ✅ OK | メッセージ送信成功 |
| MQTT Subscribe | ✅ OK | mosquitto_subで受信確認 |

### 実証済みコード（動作確認済み）

```python
import board
import busio
import digitalio
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool
import adafruit_minimqtt.adafruit_minimqtt as MQTT

# W5500初期化
spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)
cs = digitalio.DigitalInOut(board.GP17)
rst = digitalio.DigitalInOut(board.GP20)
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)
print("Pico IP:", eth.pretty_ip(eth.ip_address))

# SocketPool作成（CircuitPython 10.x方式）
pool = SocketPool(eth)

# MQTTクライアント作成・接続・Publish
mqtt = MQTT.MQTT(broker="192.168.15.14", port=1883, socket_pool=pool)
mqtt.connect()
mqtt.publish("test/pico/w5500", "Hello from W5500-EVB-Pico2!")
mqtt.disconnect()
```

### 確認コマンド（Mosquitto側）

```bash
# Subscribe（メッセージ受信確認）
docker exec arsprout-mqtt mosquitto_sub -t "test/pico/w5500" -v

# 出力例:
# test/pico/w5500 Hello from W5500-EVB-Pico2!
```

### 発見した注意点

1. **CircuitPython 10.x では SocketPool API が必須**
   - 旧API（`MQTT.set_socket()`）は使用不可
   - `adafruit_wiznet5k_socketpool` モジュールを使用

2. **ライブラリ依存関係**
   - `adafruit_minimqtt` → `adafruit_connection_manager`, `adafruit_ticks` が必要
   - circupで自動解決される

3. **board定義の確認**
   - `dir(board)` で利用可能なピン・機能を確認
   - `W5K_*` プレフィックスのピン定義あり（W5K_SPI, W5K_CS等）

4. **デバッグ時のシリアル接続**
   - `/dev/ttyACM0` (Linux)
   - `screen /dev/ttyACM0 115200` でREPL接続
   - Ctrl+C でコード停止、Ctrl+D でソフトリブート

---

## 参考リンク

- [WIZnet公式ドキュメント](https://docs.wiznet.io/Product/Chip/Ethernet/W5500/W5500-EVB-Pico-PoE)
- [WIZnet W5500-EVB-Pico 製品ページ](https://wiznet.io/products/evaluation-boards/w5500-evb-pico)
- [CircuitPython W5500 ライブラリ](https://docs.circuitpython.org/projects/wiznet5k/en/stable/)
- [Adafruit Learn: Networking with WIZnet](https://learn.adafruit.com/networking-in-circuitpython/networking-with-wiznet-ethernet)
- [GitHub: adafruit_wiznet5k](https://github.com/adafruit/Adafruit_CircuitPython_Wiznet5k)
- [Grove Shield for Pi Pico (Seeed Studio)](https://www.seeedstudio.com/Grove-Shield-for-Pi-Pico-v1-0-p-4846.html)
- [M5Stack Unit ENV III](https://docs.m5stack.com/en/unit/envIII)

## 関連スキル

- **pico-setup-wizard**: Pico系ボードのセットアップウィザード
- **circuitpython-sensor-mqtt-builder**: センサー+MQTT統合コード生成
- **circuitpython-network-manager**: ネットワーク接続管理モジュール生成

---

## 統合元からの補足: Ethernet接続テスト

### 接続テスト7段階シナリオ

W5500イーサネット接続の段階的な診断手順。各段階で期待結果を確認し、問題箇所を特定する。

#### 1. ハードウェア認識テスト
```python
import board
print("W5500 pins:", [hasattr(board, f'GP{i}') for i in [16,17,18,19,20,21]])
# 期待結果: [True, True, True, True, True, True]
```

#### 2. SPI初期化テスト
```python
spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)
cs = digitalio.DigitalInOut(board.GP17)
rst = digitalio.DigitalInOut(board.GP20)
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)
print("MAC:", [hex(i) for i in eth.mac_address])
# 期待結果: MAC: ['0xde', '0xad', '0xbe', '0xef', '0xfe', '0xed']
```

#### 3. DHCP IP取得テスト
```python
import time
time.sleep(2)  # DHCP応答待ち
print("IP:", eth.pretty_ip(eth.ip_address))
print("Link:", eth.link_status)
# 期待結果: IP: 192.168.x.x, Link: True
```

#### 4. 静的IP設定テスト（DHCP失敗時）
```python
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=False)
eth.ifconfig = (
    (192, 168, 1, 100), (255, 255, 255, 0),
    (192, 168, 1, 1), (8, 8, 8, 8)
)
print("IP:", eth.pretty_ip(eth.ip_address))
# 期待結果: IP: 192.168.1.100
```

#### 5. SocketPool API テスト（CircuitPython 10.x）
```python
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool
pool = SocketPool(eth)
print("SocketPool:", type(pool).__name__)
# 期待結果: SocketPool: SocketPool
```

#### 6. TCP疎通テスト（HTTP GET）
```python
import adafruit_requests
requests = adafruit_requests.Session(pool)
r = requests.get("http://wifitest.adafruit.com/testwifi/index.html")
print("Status:", r.status_code)
r.close()
# 期待結果: Status: 200
```

#### 7. MQTT接続テスト
```python
import adafruit_minimqtt.adafruit_minimqtt as MQTT
mqtt = MQTT.MQTT(broker="192.168.x.x", port=1883, socket_pool=pool)
mqtt.connect()
mqtt.publish("test/w5500", "Hello!")
mqtt.disconnect()
# 期待結果: MQTT Connected! → Message published → MQTT Disconnected
```

### 完全な診断スクリプト

```python
"""W5500-EVB-Pico2 接続テスト - CircuitPython 10.x"""
import board, busio, digitalio, time
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K
from adafruit_wiznet5k.adafruit_wiznet5k_socketpool import SocketPool

print("="*50)
print("W5500 Ethernet Connection Test")
print("="*50)

# [1/6] Hardware Check
print("\n[1/6] Hardware Check...")
assert all(hasattr(board, f'GP{i}') for i in [16,17,18,19,20]), "GPIO pins missing"
print("✅ Hardware pins OK")

# [2/6] SPI Initialization
print("\n[2/6] SPI Initialization...")
spi = busio.SPI(board.GP18, MOSI=board.GP19, MISO=board.GP16)
cs = digitalio.DigitalInOut(board.GP17)
rst = digitalio.DigitalInOut(board.GP20)
print("✅ SPI initialized")

# [3/6] W5500 Initialization
print("\n[3/6] W5500 Initialization...")
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)
print(f"✅ MAC: {':'.join([f'{b:02x}' for b in eth.mac_address])}")

# [4/6] Link Status
print("\n[4/6] Link Status Check...")
time.sleep(1)
if eth.link_status:
    print("✅ Ethernet link UP")
else:
    print("❌ Link DOWN - Check LAN cable")
    raise RuntimeError("Link down")

# [5/6] DHCP IP Acquisition
print("\n[5/6] DHCP IP Acquisition...")
time.sleep(2)
ip = eth.pretty_ip(eth.ip_address)
if ip != "0.0.0.0":
    print(f"✅ IP: {ip}")
else:
    print("❌ DHCP failed - Use static IP")
    raise RuntimeError("DHCP failed")

# [6/6] SocketPool Creation
print("\n[6/6] SocketPool Creation...")
pool = SocketPool(eth)
print(f"✅ SocketPool: {type(pool).__name__}")

print("\n" + "="*50)
print("✅ All tests passed!")
print(f"IP: {ip} | Link: UP")
print("="*50)
```

### 主要なトラブルシューティングパターン

| 症状 | 原因 | 診断コマンド | 対処法 |
|------|------|------------|--------|
| Link LED消灯 | ケーブル未接続 | `eth.link_status` | LANケーブル確認 |
| IP: 0.0.0.0 | DHCP失敗 | `time.sleep(3)` 後に再確認 | 静的IP設定に変更 |
| SPI通信エラー | ピン配置誤り | GPIOピン再確認 | GP16-20配線確認 |
| ImportError | ライブラリ不足 | `circup list` | `circup install adafruit_wiznet5k` |
| MQTT接続失敗 | ブローカーIP誤り | `ping <broker_ip>` | IPアドレス・ポート確認 |

### ハードウェアリセット手順

```python
# W5500が応答しない場合の強制リセット
rst.value = False
time.sleep(0.1)
rst.value = True
time.sleep(0.5)
eth = WIZNET5K(spi, cs, reset=rst, is_dhcp=True)
```

### CircuitPython 10.x API 移行チェックリスト

- [ ] `adafruit_wiznet5k_socketpool` をインポート（旧: `adafruit_wiznet5k_socket`）
- [ ] `SocketPool(eth)` でプール作成
- [ ] `MQTT.MQTT(socket_pool=pool)` を使用（旧: `MQTT.set_socket()`）
- [ ] `adafruit_requests.Session(pool)` を使用（旧: `adafruit_requests.set_socket()`）
