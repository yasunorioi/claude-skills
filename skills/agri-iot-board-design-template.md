# Agricultural IoT Board Design Template - Skill Definition

**Skill ID**: `agri-iot-board-design-template`
**Category**: Hardware Design / PCB Design
**Version**: 1.0.0
**Created**: 2026-02-05
**Platform**: JLCPCB / KiCad

---

## Overview

This skill provides a comprehensive template for designing custom PCBs for agricultural IoT nodes. It covers Grove Shield alternative designs, relay control circuits, I2C bus extension with P82B96, and JLCPCB-ready component selection with LCSC part numbers.

---

## Use Cases

- **Greenhouse Sensor Node**: Temperature, humidity, CO2 monitoring with solenoid valve control
- **Irrigation Controller**: Soil moisture sensing with single valve control
- **Environmental Monitoring**: Multi-sensor nodes with long cable runs (5-20m)
- **Retrofit Projects**: Adding IoT capability to existing greenhouse equipment

---

## Target Configuration

### 構成A: フル構成（基板にI2Cバッファ内蔵）
```
W5500-EVB-Pico-PoE (Base Board)
       │
       └── Custom Expansion Board
           ├── Grove Connectors (I2C x2, ADC x1)
           ├── Relay 1ch (Solenoid Valve Control)
           ├── P82B96 I2C Buffer (5-20m Cable Extension)
           ├── Screw Terminals (Field Wiring)
           └── Status LEDs
```

### 構成B: シンプル構成（I2Cハブ外付け）★推奨
```
W5500-EVB-Pico-PoE (Base Board)
       │
       └── Custom Expansion Board (Minimal)
           ├── Grove I2C x1 ← 1ポートのみ！
           ├── Relay 1ch
           └── Status LED
                │
                └── 外部 I2Cハブ (TCA9548A等)
                    ├── センサー1 (SHT40)
                    ├── センサー2 (SCD41)
                    ├── センサー3 (BMP280)
                    └── P82B96 → 長距離センサー
```

**構成Bのメリット**:
- 基板がシンプル（部品点数削減、コスト↓）
- I2Cハブで同一アドレスセンサー複数接続可能
- ユーザーが好みのハブを選択可能
- 失敗時の不良在庫リスク低減

---

## Design Sections

### 1. Grove Shield Alternative Design

#### Purpose

Replace discontinued Grove Shield for Pi Pico with custom PCB for production.

#### Grove Connector Pinout (HY2.0-4P)

| Pin | I2C Function | ADC Function |
|-----|--------------|--------------|
| 1   | GND          | GND          |
| 2   | VCC (3.3V)   | VCC (3.3V)   |
| 3   | SDA          | NC           |
| 4   | SCL          | Signal       |

#### Recommended Layout

```
┌─────────────────────────────────────────┐
│  [Grove I2C-1]  [Grove I2C-2]  [Grove ADC] │
│      │              │              │      │
│      └──────────────┴──────────────┘      │
│                     │                      │
│              ┌──────┴──────┐               │
│              │   P82B96    │               │
│              │ I2C Buffer  │               │
│              └──────┬──────┘               │
│                     │                      │
│  [EXT I2C] ←────────┘      [Relay 1ch]    │
│                                  │         │
│  [Terminal Block] ←──────────────┘         │
│                                            │
│  ════════════════════════════════════      │
│  │ Pin Header for W5500-EVB-Pico-PoE │     │
│  ════════════════════════════════════      │
└─────────────────────────────────────────┘
```

---

### 2. P82B96 I2C Bus Buffer Circuit

#### Overview

The P82B96 is a dual bidirectional I2C bus buffer that enables cable extension up to 20+ meters while maintaining 400kHz Fast I2C operation.

#### Key Specifications

| Parameter | Main Side (Sx/Sy) | Transmission Side (Tx/Ty) |
|-----------|-------------------|---------------------------|
| Max Capacitance | 400 pF | 4000 pF |
| Voltage Range | 2V - 15V | 2V - 15V |
| Max Cable Length | - | 20+ meters |
| I2C Speed | 400 kHz (Fast mode) | 400 kHz |

#### Circuit Diagram

```
                      P82B96
                   ┌─────────┐
    Pico SDA ──────┤ Sx   Tx ├──────┬───── EXT_SDA (to remote sensor)
                   │         │      │
    Pico SCL ──────┤ Sy   Ty ├──────┼───── EXT_SCL (to remote sensor)
                   │         │      │
         VCC ──────┤ VCC     │      │
                   │         │      │
         GND ──────┤ GND     │      │
                   └─────────┘      │
                                    │
              Local Side          Remote Side
              (< 0.5m)           (5-20m cable)

    Pull-up Resistors:
    ────────────────────────────────────────────
    Pico SDA ──[4.7kΩ]── VCC    EXT_SDA ──[2.2kΩ]── VCC
    Pico SCL ──[4.7kΩ]── VCC    EXT_SCL ──[2.2kΩ]── VCC
```

#### Design Notes

1. **Pull-up Resistors**
   - Local side (Sx/Sy): 4.7kΩ (standard I2C)
   - Remote side (Tx/Ty): 2.2kΩ (stronger pull-up for long cables)

2. **Cable Requirements**
   - Use twisted pair cable (Cat5e or shielded)
   - SDA and SCL on separate twisted pairs
   - Connect shield to GND at one end only

3. **推奨ケーブル: M5Stack GROVE互換ケーブル**
   - スイッチサイエンスで購入可能
   - 100cm (¥550): https://www.switch-science.com/products/5216
   - 200cm: https://www.switch-science.com/products/5217
   - Grove端子付きでそのまま接続可能
   - P82B96と併用で5〜20m延長OK

3. **Decoupling Capacitors**
   - Add 100nF ceramic capacitor near P82B96 VCC pin
   - Add 100nF at remote sensor VCC

#### P82B96 Pin Configuration (SOIC-8)

| Pin | Name | Function |
|-----|------|----------|
| 1   | Sx   | SDA (local side) |
| 2   | Sy   | SCL (local side) |
| 3   | Ty   | SCL (transmission/remote side) |
| 4   | GND  | Ground |
| 5   | Tx   | SDA (transmission/remote side) |
| 6   | NC   | No connection |
| 7   | NC   | No connection |
| 8   | VCC  | Power supply (3.3V or 5V) |

---

### 3. 5V DCファン制御回路（強制換気用）

#### 用途

温湿度センサー用の強制換気ファン（5V DC）を制御。
PoEから5V供給可能なため、追加電源不要。

#### 回路図

```
GPIO (GP22) ──[1kΩ]──┬── Gate
                      │
                    ┌─┴─┐
                    │   │ 2N7002 (N-ch MOSFET)
                    └─┬─┘
                      │ Drain
                      │
              ファン (-) ──┘

              ファン (+) ── 5V (VBUS from PoE)
                      │
                  Source ── GND
```

#### 部品選定

| Component | Value | LCSC Part # | Purpose |
|-----------|-------|-------------|---------|
| MOSFET | 2N7002 (SOT-23) | C8545 | ローサイドスイッチ |
| Gate Resistor | 1kΩ 0603 | C22548 | 電流制限 |
| Fan Connector | 2.54mm 2P | C395881 | ファン接続端子 |

#### 仕様

| Parameter | Value |
|-----------|-------|
| 制御電圧 | 3.3V GPIO |
| ファン電圧 | 5V (VBUS) |
| 最大電流 | 500mA（2N7002定格） |
| 制御方式 | ON/OFF or PWM |

#### CircuitPython 制御コード

```python
import board
import digitalio
import pwmio
import time

# ========================================
# 方式1: ON/OFF制御（シンプル）
# ========================================
fan = digitalio.DigitalInOut(board.GP22)
fan.direction = digitalio.Direction.OUTPUT

def fan_on():
    fan.value = True

def fan_off():
    fan.value = False

# ========================================
# 方式2: PWM速度制御（風量調整可能）
# ========================================
fan_pwm = pwmio.PWMOut(board.GP22, frequency=25000, duty_cycle=0)

def set_fan_speed(percent):
    """ファン速度を0-100%で設定"""
    duty = int(percent / 100 * 65535)
    fan_pwm.duty_cycle = duty

# 使用例
set_fan_speed(50)   # 50%
set_fan_speed(100)  # 全開
set_fan_speed(0)    # 停止
```

#### 温湿度連動制御（実用例）

```python
import board
import busio
import digitalio
import adafruit_sht4x
import time

# センサー初期化
i2c = busio.I2C(scl=board.GP5, sda=board.GP4)
sht = adafruit_sht4x.SHT4x(i2c)

# ファン初期化
fan = digitalio.DigitalInOut(board.GP22)
fan.direction = digitalio.Direction.OUTPUT

# 制御パラメータ
FAN_ON_HUMIDITY = 80    # 湿度80%以上でファンON
FAN_OFF_HUMIDITY = 70   # 湿度70%以下でファンOFF
FAN_ON_TEMP = 35        # 温度35℃以上でファンON
FAN_OFF_TEMP = 30       # 温度30℃以下でファンOFF

def control_fan(temp, humidity):
    """温湿度に基づくファン制御（ヒステリシス付き）"""
    if humidity > FAN_ON_HUMIDITY or temp > FAN_ON_TEMP:
        fan.value = True
        return "ON"
    elif humidity < FAN_OFF_HUMIDITY and temp < FAN_OFF_TEMP:
        fan.value = False
        return "OFF"
    # 範囲内なら現状維持
    return "ON" if fan.value else "OFF"

# メインループ
while True:
    temperature, humidity = sht.measurements
    status = control_fan(temperature, humidity)
    print(f"Temp: {temperature:.1f}°C, Humid: {humidity:.1f}%, Fan: {status}")
    time.sleep(10)
```

#### MQTT連携（Home Assistant制御）

```python
import board
import digitalio
import json
from adafruit_minimqtt import adafruit_minimqtt as MQTT

# ファン初期化
fan = digitalio.DigitalInOut(board.GP22)
fan.direction = digitalio.Direction.OUTPUT

HOUSE_ID = "h1"
FAN_COMMAND_TOPIC = f"greenhouse/{HOUSE_ID}/fan/set"
FAN_STATE_TOPIC = f"greenhouse/{HOUSE_ID}/fan/state"

def on_fan_command(client, topic, message):
    """Home Assistantからのファン制御コマンド受信"""
    payload = message.upper()
    if payload == "ON" or payload == "1":
        fan.value = True
        client.publish(FAN_STATE_TOPIC, "ON")
    elif payload == "OFF" or payload == "0":
        fan.value = False
        client.publish(FAN_STATE_TOPIC, "OFF")

# MQTTクライアント設定
mqtt_client.subscribe(FAN_COMMAND_TOPIC)
mqtt_client.on_message = on_fan_command

# Home Assistant MQTT Discovery（自動登録）
discovery_payload = {
    "name": "ハウス1 換気ファン",
    "command_topic": FAN_COMMAND_TOPIC,
    "state_topic": FAN_STATE_TOPIC,
    "unique_id": f"{HOUSE_ID}_fan",
    "device": {
        "identifiers": [f"pico_{HOUSE_ID}"],
        "name": f"Greenhouse {HOUSE_ID}",
        "manufacturer": "DIY"
    }
}
mqtt_client.publish(
    f"homeassistant/switch/{HOUSE_ID}_fan/config",
    json.dumps(discovery_payload),
    retain=True
)
```

---

### 4. ECセンサー用ノイズ対策回路

#### 課題

ECセンサーは微小電流を測定するため、PoEのスイッチングノイズが測定精度に直接影響する。
MCP3421（18bit ADC）の精度を活かすには、電源のクリーン化が必須。

#### ノイズ対策構成

```
PoE 5V (ノイズあり)
    │
    ▼
┌─────────────┐
│ LDO         │ AMS1117-3.3 or XC6206
│ (5V→3.3V)   │ ← スイッチングノイズを除去
└──────┬──────┘
       │
    100µF (電解)
       │
    0.1µF (セラミック)
       │
       ▼
┌─────────────┐
│ フェライト   │ BLM18AG601SN1
│ ビーズ       │ ← 高周波ノイズ除去
└──────┬──────┘
       │
    0.1µF
       │
       ├──── MCP3421 VDD
       │
       └──── ECフロントエンド電源
```

#### 部品選定

| Component | Value | LCSC Part # | Purpose |
|-----------|-------|-------------|---------|
| LDO | AMS1117-3.3 | C6186 | 5V→3.3V変換、ノイズ除去 |
| LDO (低ドロップ) | XC6206P332MR | C5446 | 省電力向け |
| 電解コンデンサ | 100µF/16V | C440199 | 低周波リップル除去 |
| セラミックC | 0.1µF 0603 | C14663 | 高周波デカップリング |
| フェライトビーズ | BLM18AG601 | C71858 | 高周波ノイズ抑制 |

#### EC計測専用基板（3ライン製品の1つ）

EC計測に特化した構成。ノイズ対策が肝。

```
W5500-EVB-Pico-PoE
       │
       └── EC Sensor Board (Minimal)
           ├── Grove I2C x1 → MCP3421 + ECフロントエンド
           ├── ノイズ対策電源 (LDO + フェライト)
           └── Status LED
```

**3ライン製品ラインナップ**:

| 製品 | 用途 | 特徴 |
|------|------|------|
| **センサーノード** | 温湿度/CO2計測 | Grove I2C + I2Cハブ + 5Vファンコネクタ |
| **リレーノード** | 電磁弁/ポンプ等 | 既製品活用（Waveshare Pico-Relay-B等） |
| **ECセンサーノード** | EC計測専用 | ノイズ対策電源 + MCP3421 |

#### センサーノード詳細

```
W5500-EVB-Pico-PoE
       │
       └── Sensor Node Board
           ├── Grove I2C x1 → I2Cハブ経由でセンサー接続
           ├── 5V ファンコネクタ (2P) → 強制換気用
           └── Status LED
```

#### リレーノード（既製品活用）

自作せず、既製品を活用する。GPIO→ターミナルブロック出しの汎用製品。

| 製品 | ch数 | 定格 | 価格 | 特徴 |
|------|------|------|------|------|
| **★ 8ch絶縁リレーモジュール PoE版** | 8ch | 250VAC/10A | ¥8,371 | **RP2350+W5500+PoE+RS485 全部入り** |
| Waveshare Pico-Relay-B | 8ch | 250VAC/10A | $15程度 | フォトカプラ絶縁、Pico別途必要 |
| PicoRelay8 | 8ch | 250VAC/10A | $12程度 | シンプル、Pico別途必要 |

**★ 最推奨: 8ch絶縁入力リレーモジュール（RP2350搭載、PoE版）**
- https://www.switch-science.com/products/10824
- RP2350B（デュアルCortex-M33、150MHz）搭載
- W5500 + PoE 内蔵 → 別途Picoボード不要
- 8ch絶縁デジタル入力（5〜36V対応）
- 8chリレー出力（10A/250VAC）
- RS-485ポート搭載 → 外気象センサー接続可能
- RTC、TFカード、USB Type-C
- DINレール対応
- **これ1台で制御ノード完結！**

※ スイッチサイエンスで販売中（在庫確認要）

**Waveshare Pico-Relay-B**（代替）:
- https://www.waveshare.com/pico-relay-b.htm
- Pico直挿し、8chリレー全てターミナルブロック出し
- フォトカプラ絶縁で安全
- 農業用途: 電磁弁、ポンプ、換気扇、暖房機等
- **注意**: W5500-EVB-Pico-PoEとは直接スタックできない
  → Pico 2 W + WiFi 構成で使用

#### ECフロントエンド回路（参考）

```
          ┌────────────────────────────────────────┐
          │       ECプローブ                        │
          │         │                              │
          │    ┌────┴────┐                         │
          │    │ 測定抵抗 │ 100Ω〜10kΩ（レンジ切替）│
          │    └────┬────┘                         │
          │         │                              │
          │    ┌────┴────┐                         │
          │    │LM358    │ オペアンプ（増幅）        │
          │    │(C7950)  │ $0.07                   │
          │    └────┬────┘                         │
          │         │                              │
          │    ┌────┴────┐                         │
          │    │MCP3421  │ 18bit I2C ADC           │
          │    │(C29454) │ $1.39                   │
          │    └────┬────┘                         │
          │         │ I2C                          │
          └─────────┴──────────────────────────────┘
                    │
                 To Pico
```

**参考**: https://jitaku-yasai.com/home-made/ec-meter-selfmade/

#### ECセンサーノード構成

```
W5500-EVB-Pico-PoE
       │
       └── EC Sensor Board
           ├── MCP3421 + ECフロントエンド
           ├── ノイズ対策電源 (LDO + フェライト)
           ├── [CAL] ボタン ← キャリブレーション
           ├── [MODE] ボタン ← モード切替
           └── Status LED
```

#### キャリブレーション操作

```
【2ボタン操作】
  [CAL] ← キャリブレーション  [MODE] ← モード切替/キャンセル

【手順】
1. プローブを校正液1（1.413 mS/cm）に浸す
2. [CAL] 長押し3秒 → LED点滅（キャリブモード開始）
3. 値が安定したら [CAL] 短押し → LED1回点灯（校正点1保存）
4. プローブを洗浄 → 校正液2（12.88 mS/cm）に浸す
5. 値が安定したら [CAL] 短押し → LED2回点灯（校正点2保存）
6. [MODE] 押す or 30秒放置 → 通常モードに戻る

【LED表示】
- 消灯: 通常動作中
- 点滅: キャリブモード（入力待ち）
- 1回点灯: 校正点1完了
- 2回点灯: 校正点2完了
- 高速点滅: エラー（値が範囲外等）

【校正液】
- 低レンジ: 1.413 mS/cm（標準）
- 高レンジ: 12.88 mS/cm（標準）
```

#### キャリブレーションデータ保存（Flash）

```python
import json
import microcontroller

# キャリブレーションデータ構造
cal_data = {
    "cal_low": 1.413,      # 校正液1 (mS/cm)
    "raw_low": 12345,      # 校正液1のADC生値
    "cal_high": 12.88,     # 校正液2 (mS/cm)
    "raw_high": 54321,     # 校正液2のADC生値
    "calibrated_at": "2026-02-06T12:00:00"
}

# Flash NVM に保存（Pico内蔵）
def save_calibration(cal_data):
    data_bytes = json.dumps(cal_data).encode('utf-8')
    # NVMは最大4096バイト
    microcontroller.nvm[0:len(data_bytes)] = data_bytes
    microcontroller.nvm[len(data_bytes)] = 0  # 終端

def load_calibration():
    try:
        end = microcontroller.nvm[:].index(0)
        data_bytes = bytes(microcontroller.nvm[0:end])
        return json.loads(data_bytes.decode('utf-8'))
    except:
        return None  # 未キャリブレーション

# EC値計算（2点校正の線形補間）
def calc_ec(raw_value, cal_data):
    if cal_data is None:
        return None  # 要キャリブレーション

    slope = (cal_data["cal_high"] - cal_data["cal_low"]) / \
            (cal_data["raw_high"] - cal_data["raw_low"])
    ec = cal_data["cal_low"] + slope * (raw_value - cal_data["raw_low"])
    return ec
```

#### BOM（ECセンサー基板）

| Component | Value | LCSC Part # | Qty | Price |
|-----------|-------|-------------|-----|-------|
| LDO | AMS1117-3.3 | C6186 | 1 | $0.10 |
| Op-Amp | LM358DR2G | C7950 | 1 | $0.07 |
| ADC | MCP3421 | C29454 | 1 | $1.39 |
| 電解C | 100µF/16V | C440199 | 1 | $0.03 |
| セラミックC | 0.1µF 0603 | C14663 | 4 | $0.004 |
| フェライト | BLM18AG601 | C71858 | 1 | $0.02 |
| 測定抵抗 | 1kΩ 0603 | C22548 | 1 | $0.001 |
| Grove端子 | HY2.0-4P | C722729 | 1 | $0.04 |
| タクトスイッチ | 6x6mm | C318884 | 2 | $0.02 |
| LED | 0603 Green | C125098 | 1 | $0.01 |
| | | | **Subtotal** | **$1.69** |

※ MCP3421は Extended Parts のため $3 手数料追加の可能性あり

---

### 5. 共通Web UI + ログ + MQTT異常通知

#### Web UIエンドポイント（全ノード共通）

```
http://[ノードIP]/
├── /         ← メイン画面（センサー値/状態）
├── /log      ← ログ閲覧（直近100件 + 異常履歴）
└── /config   ← 設定確認（読み取り専用）
```

#### 共通HTMLテンプレート

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>{node_name}</title>
  <style>
    body { font-family: sans-serif; margin: 0; padding: 16px; background: #1a1a2e; color: #fff; }
    .header { display: flex; justify-content: space-between; margin-bottom: 16px; }
    .status { padding: 4px 8px; border-radius: 4px; }
    .ok { background: #10b981; }
    .warn { background: #f59e0b; }
    .error { background: #ef4444; }
    .card { background: #16213e; padding: 16px; border-radius: 8px; margin-bottom: 12px; }
    .value { font-size: 48px; font-weight: bold; }
    .unit { font-size: 18px; color: #888; }
    .label { color: #888; margin-bottom: 4px; }
    .nav { display: flex; gap: 8px; margin-top: 16px; }
    .nav a { color: #60a5fa; text-decoration: none; }
    .log { font-family: monospace; font-size: 12px; white-space: pre; overflow-x: auto; }
  </style>
</head>
<body>
  <div class="header">
    <div><strong>{node_name}</strong> ({node_type})</div>
    <div class="status {status_class}">{status}</div>
  </div>
  {content}
  <div class="nav">
    <a href="/">Home</a> | <a href="/log">Log</a> | <a href="/config">Config</a>
  </div>
  <div style="color:#666;margin-top:16px;font-size:12px;">
    IP: {ip} | Uptime: {uptime} | MQTT: {mqtt_status}
  </div>
</body>
</html>
```

#### ノード別メインコンテンツ

**センサーノード**:
```html
<div class="card">
  <div class="label">温度</div>
  <div class="value">25.3<span class="unit">℃</span></div>
</div>
<div class="card">
  <div class="label">湿度</div>
  <div class="value">65<span class="unit">%</span></div>
</div>
<div class="card">
  <div class="label">CO2</div>
  <div class="value">850<span class="unit">ppm</span></div>
</div>
```

**ECノード**:
```html
<div class="card">
  <div class="label">EC値</div>
  <div class="value">2.4<span class="unit">mS/cm</span></div>
</div>
<div class="card">
  <div class="label">キャリブレーション</div>
  <div>最終: 2026-02-06 12:00</div>
  <div class="status ok">正常</div>
</div>
```

#### ログ設計

**通常ログ（リングバッファ、直近100件）**:
```json
{"ts": 1707200000, "temp": 25.3, "humid": 65, "co2": 850}
{"ts": 1707200060, "temp": 25.4, "humid": 64, "co2": 855}
```

**異常ログ（別枠保持、最大50件）**:
```json
{"ts": 1707200000, "type": "sensor_error", "msg": "SHT40 read failed"}
{"ts": 1707200100, "type": "mqtt_disconnect", "msg": "Broker unreachable"}
{"ts": 1707200200, "type": "calibration", "msg": "EC calibration completed"}
{"ts": 1707200300, "type": "reboot", "msg": "Watchdog reset"}
```

#### CircuitPythonログ実装

```python
import json
import time
import microcontroller

# ログ設定
MAX_NORMAL_LOGS = 100
MAX_ERROR_LOGS = 50
LOG_FILE = "/logs/normal.json"
ERROR_FILE = "/logs/error.json"

# メモリ内バッファ（起動時にFlashから読み込み）
normal_logs = []
error_logs = []

def add_log(data):
    """通常ログ追加（リングバッファ）"""
    global normal_logs
    data["ts"] = time.time()
    normal_logs.append(data)
    if len(normal_logs) > MAX_NORMAL_LOGS:
        normal_logs.pop(0)

def add_error(error_type, message):
    """異常ログ追加"""
    global error_logs
    entry = {"ts": time.time(), "type": error_type, "msg": message}
    error_logs.append(entry)
    if len(error_logs) > MAX_ERROR_LOGS:
        error_logs.pop(0)
    # 異常は即座にFlash保存
    save_logs()

def save_logs():
    """ログをFlashに保存（定期的 or 異常時）"""
    try:
        with open(LOG_FILE, "w") as f:
            json.dump(normal_logs[-50:], f)  # 直近50件のみ保存
        with open(ERROR_FILE, "w") as f:
            json.dump(error_logs, f)
    except OSError:
        pass  # Flash書き込みエラーは無視

def load_logs():
    """起動時にFlashからログ読み込み"""
    global normal_logs, error_logs
    try:
        with open(LOG_FILE, "r") as f:
            normal_logs = json.load(f)
    except:
        normal_logs = []
    try:
        with open(ERROR_FILE, "r") as f:
            error_logs = json.load(f)
    except:
        error_logs = []
```

#### MQTT異常通知トピック

```
greenhouse/{house_id}/node/{node_id}/
├── status          # "online" / "offline" (LWT)
├── error           # 異常発生時にpublish
└── heartbeat       # 定期死活監視（60秒間隔）

# 異常通知ペイロード例
{
  "type": "sensor_error",
  "sensor": "SHT40",
  "message": "I2C communication failed",
  "timestamp": 1707200000
}
```

**異常タイプ一覧**:

| type | 内容 | 重要度 |
|------|------|--------|
| `sensor_error` | センサー読み取り失敗 | 高 |
| `sensor_range` | 値が範囲外 | 中 |
| `mqtt_disconnect` | MQTT切断 | 高 |
| `calibration_needed` | 要キャリブレーション | 中 |
| `low_memory` | メモリ不足警告 | 低 |
| `reboot` | 再起動発生 | 中 |

#### Home Assistant MQTT Discovery（異常通知）

```python
# 異常通知用センサー自動登録
discovery_payload = {
    "name": f"{HOUSE_ID} {NODE_ID} Error",
    "state_topic": f"greenhouse/{HOUSE_ID}/node/{NODE_ID}/error",
    "value_template": "{{ value_json.type }}",
    "json_attributes_topic": f"greenhouse/{HOUSE_ID}/node/{NODE_ID}/error",
    "unique_id": f"{HOUSE_ID}_{NODE_ID}_error",
    "device_class": "problem",
    "device": {
        "identifiers": [f"pico_{HOUSE_ID}_{NODE_ID}"],
        "name": f"Node {NODE_ID}",
        "manufacturer": "DIY"
    }
}
mqtt_client.publish(
    f"homeassistant/sensor/{HOUSE_ID}_{NODE_ID}_error/config",
    json.dumps(discovery_payload),
    retain=True
)
```

---

### 6. Relay Driver Circuit (1 Channel)

#### Circuit Diagram

```
    GPIO Pin (3.3V) ──[1kΩ]──┬── Base
                             │
                           ┌─┴─┐
                           │   │ 2N2222A (NPN)
                           └─┬─┘
                             │ Collector
                             │
                         ┌───┴───┐
                         │       │
                         │ RELAY │ SRD-05VDC-SL-C
                         │  (5V) │
                         │       │
                         └───┬───┘
                             │ Coil+
                             │
         5V ────────────┬────┘
                        │
                   ┌────┴────┐
                   │ 1N4148  │ Flyback Diode
                   │  ◀──    │ (cathode to 5V)
                   └────┬────┘
                        │
                        └──── Coil-
                             │
         GND ────────────────┴──── Emitter


    LED Indicator:
    ─────────────────────────────────────
    Relay NO ──[330Ω]──[LED]── GND
```

#### Component Selection

| Component | Value | Purpose |
|-----------|-------|---------|
| Base Resistor | 1kΩ | Limit base current (~3mA) |
| Flyback Diode | 1N4148 | Suppress back-EMF |
| LED Resistor | 330Ω | Limit LED current (~10mA) |
| Relay | 5V SPDT | Solenoid valve control |

#### Relay Specifications (SRD-05VDC-SL-C)

| Parameter | Value |
|-----------|-------|
| Coil Voltage | 5V DC |
| Coil Current | ~70mA |
| Contact Rating | 10A @ 250VAC |
| Contact Type | SPDT (NO/NC/COM) |

#### Screw Terminal Wiring

```
    ┌─────────────────┐
    │ 1: COM          │ → Common (valve power)
    │ 2: NO           │ → Normally Open (valve+)
    │ 3: NC           │ → Normally Closed (unused)
    └─────────────────┘
```

---

### 7. JLCPCB Component Selection (LCSC Part Numbers)

#### Bill of Materials (BOM)

| Component | Description | LCSC Part # | Qty | Unit Price | Extended |
|-----------|-------------|-------------|-----|------------|----------|
| **Connectors** |
| Grove Connector | HY2.0-4P SMD | C722729 | 3 | $0.04 | $0.12 |
| Pin Header | 2.54mm 2x20P | C50980 | 1 | $0.08 | $0.08 |
| Screw Terminal | 5mm 2P | C395868 | 2 | $0.09 | $0.18 |
| **I2C Buffer** |
| P82B96TD,118 | I2C Buffer SOIC-8 | C32103 | 1 | $1.50 | $1.50 |
| **Relay Circuit** |
| SRD-05VDC-SL-C | 5V Relay SPDT | C35449 | 1 | $0.26 | $0.26 |
| 2N2222A | NPN Transistor SOT-23 | C118536 | 1 | $0.02 | $0.02 |
| 1N4148 | Flyback Diode SOD-323 | C14516 | 1 | $0.004 | $0.004 |
| **Passives** |
| Resistor 1kΩ | 0603 1% | C22548 | 1 | $0.001 | $0.001 |
| Resistor 330Ω | 0603 1% | C23138 | 1 | $0.001 | $0.001 |
| Resistor 4.7kΩ | 0603 1% | C23162 | 2 | $0.001 | $0.002 |
| Resistor 2.2kΩ | 0603 1% | C22975 | 2 | $0.001 | $0.002 |
| Capacitor 100nF | 0603 X7R | C14663 | 3 | $0.001 | $0.003 |
| **Indicators** |
| LED Green | 0603 SMD | C125098 | 2 | $0.01 | $0.02 |
| | | | **Subtotal** | | **$2.19** |

#### PCB Manufacturing Cost (JLCPCB)

| Quantity | PCB Cost | Assembly | Total/Board |
|----------|----------|----------|-------------|
| 5 pcs | $2.00 | $8.00 + parts | ~$4.20 |
| 10 pcs | $2.00 | $8.00 + parts | ~$3.22 |
| 50 pcs | $5.00 | $8.00 + parts | ~$2.45 |

**Note**: Assembly cost is $8.00 setup + $0.0017/joint. Extended parts (P82B96) may add $3 handling fee.

#### Total Cost Estimate (Per Board)

| Quantity | Components | PCB + Assembly | Total |
|----------|------------|----------------|-------|
| 5 pcs | $2.19 | $2.80 | **$4.99** |
| 10 pcs | $2.19 | $1.30 | **$3.49** |
| 50 pcs | $2.19 | $0.36 | **$2.55** |

---

### 8. KiCad Design Reference

#### Schematic Blocks

```
Sheet 1: Power & Connectors
─────────────────────────────────
┌─────────────┐
│ Pin Header  │──── 3.3V ────┬─── Grove VCC
│ (Pico)      │              │
│             │──── 5V  ─────┼─── Relay VCC
│             │              │
│             │──── GND ─────┴─── Common GND
│             │
│             │──── SDA ─────────┐
│             │                  │
│             │──── SCL ─────────┼─── To I2C Buffer
│             │                  │
│             │──── GPIO ────────┼─── To Relay Driver
└─────────────┘                  │
                                 │
Sheet 2: I2C Buffer              │
─────────────────────────────────
┌─────────────┐                  │
│   P82B96    │◀─────────────────┘
│             │
│ Sx ──── Tx  │──── EXT_SDA
│ Sy ──── Ty  │──── EXT_SCL
│             │
└─────────────┘

Sheet 3: Relay Driver
─────────────────────────────────
GPIO ──[R1]── 2N2222A ──┬── Relay ──┬── Terminal
                        │           │
                     1N4148        LED
```

#### Recommended PCB Size

| Parameter | Value |
|-----------|-------|
| Width | 50mm |
| Height | 40mm |
| Layers | 2 |
| Thickness | 1.6mm |
| Copper | 1oz |

#### Design Rules (JLCPCB Standard)

| Parameter | Minimum |
|-----------|---------|
| Trace Width | 0.127mm (5mil) |
| Trace Spacing | 0.127mm (5mil) |
| Via Diameter | 0.3mm |
| Via Drill | 0.2mm |
| Silkscreen | 0.15mm line, 0.8mm text |

---

## Circuit Block Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                   Custom Expansion Board                        │
│                                                                │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                    │
│  │Grove I2C│    │Grove I2C│    │Grove ADC│                    │
│  │   #1    │    │   #2    │    │   #1    │                    │
│  └────┬────┘    └────┬────┘    └────┬────┘                    │
│       │              │              │                          │
│       └──────┬───────┘              │                          │
│              │ I2C Bus              │ ADC                      │
│              ▼                      ▼                          │
│       ┌──────────┐           ┌──────────┐                     │
│       │  P82B96  │           │  Direct  │                     │
│       │  Buffer  │           │  to Pico │                     │
│       └────┬─────┘           └──────────┘                     │
│            │                                                   │
│            ▼ Extended I2C                                      │
│       ┌──────────┐                                            │
│       │ Terminal │ → To remote sensor (5-20m)                 │
│       │  Block   │                                            │
│       └──────────┘                                            │
│                                                                │
│       ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│       │   GPIO   │───▶│  Relay   │───▶│ Terminal │           │
│       │ (Pico)   │    │  Driver  │    │  Block   │           │
│       └──────────┘    └──────────┘    └──────────┘           │
│                              │                                 │
│                              ▼                                 │
│                        ┌──────────┐                           │
│                        │ Status   │                           │
│                        │   LED    │                           │
│                        └──────────┘                           │
│                                                                │
│  ══════════════════════════════════════════════════════       │
│  ║  2x20 Pin Header (connects to W5500-EVB-Pico-PoE)  ║       │
│  ══════════════════════════════════════════════════════       │
└────────────────────────────────────────────────────────────────┘
```

---

## Power Distribution

```
     PoE (via W5500-EVB-Pico-PoE)
              │
              ▼
    ┌─────────────────┐
    │   W5500-EVB     │
    │   Power Rails   │
    └────────┬────────┘
             │
     ┌───────┼───────┐
     │       │       │
     ▼       ▼       ▼
   3.3V     5V     GND
     │       │       │
     │       ├───────┼─── Relay Coil (5V)
     │       │       │
     ├───────┼───────┼─── P82B96 VCC (3.3V or 5V)
     │       │       │
     └───────┼───────┼─── Grove Connectors (3.3V)
             │       │
             └───────┴─── Remote Sensor (via cable)
```

---

## Implementation Checklist

### Design Phase
- [ ] Create KiCad schematic with all blocks
- [ ] Assign LCSC footprints to all components
- [ ] Generate netlist and run ERC
- [ ] Layout PCB (place connectors first, then ICs)
- [ ] Run DRC with JLCPCB rules
- [ ] Generate Gerber files

### Ordering Phase
- [ ] Upload Gerber to JLCPCB
- [ ] Select 2-layer, 1.6mm, green solder mask
- [ ] Enable SMT assembly
- [ ] Upload BOM (LCSC format)
- [ ] Upload CPL (component placement)
- [ ] Review assembly preview
- [ ] Place order

### Testing Phase
- [ ] Visual inspection of assembled board
- [ ] Continuity test (power rails)
- [ ] I2C scan (local Grove connectors)
- [ ] I2C scan (extended bus via P82B96)
- [ ] Relay test (GPIO toggle)
- [ ] Full integration test with sensors

---

## Common Issues and Solutions

### 1. I2C Communication Fails Over Long Cable

**Symptoms**: I2C scan returns no devices on extended bus

**Solutions**:
- Verify P82B96 orientation (pin 1 marker)
- Check pull-up resistors on both sides
- Reduce I2C speed to 100kHz
- Use shielded twisted pair cable
- Check cable length (max 20m recommended)

### 2. Relay Does Not Activate

**Symptoms**: GPIO high but relay stays off

**Solutions**:
- Verify 5V supply to relay
- Check transistor orientation (EBC pinout varies)
- Measure base resistor value
- Test relay coil directly with 5V

### 3. Sensor Readings Unstable

**Symptoms**: ADC or I2C values fluctuate

**Solutions**:
- Add decoupling capacitors near sensors
- Separate analog and digital grounds
- Use star grounding topology
- Shield cables from noise sources

---

## I2Cハブ活用（構成B向け）

### I2Cハブ製品比較

| 製品 | チップ | ポート数 | 価格 | 特徴 |
|------|--------|---------|------|------|
| Grove I2C Hub | - | 4ポート | ¥300程度 | 単純分岐（アドレス重複不可） |
| **TCA9548A** | TCA9548A | **8ch** | ¥500程度 | **アドレス重複OK**、M5Stackあり |
| PCA9548A | PCA9548A | 8ch | 同上 | TCA互換 |

### TCA9548A の威力

**同一I2Cアドレスのセンサーを複数接続可能！**

```python
import board
import busio
from adafruit_tca9548a import TCA9548A

i2c = busio.I2C(scl=board.GP5, sda=board.GP4)
tca = TCA9548A(i2c)  # アドレス 0x70

# チャンネル0: SHT40 (0x44) - ハウス内
sht_inside = adafruit_sht4x.SHT4x(tca[0])

# チャンネル1: SHT40 (0x44) - ハウス外
sht_outside = adafruit_sht4x.SHT4x(tca[1])

# → 同じ0x44でも別チャンネルなのでOK！
```

### M5Stack製品

| 製品名 | 型番 | 価格 | URL |
|--------|------|------|-----|
| I2C Hub 1 to 6 | U006 | ¥330 | https://www.switch-science.com/products/5765 |
| PaHub2 (TCA9548A) | U040-B | ¥990 | https://www.switch-science.com/products/9517 |

### 構成B 回路図

```
Pico I2C (GP4/GP5)
       │
       └── Grove端子 (基板上、1ポート)
              │
              └── Groveケーブル (20cm)
                     │
              ┌──────┴──────┐
              │  TCA9548A   │  ← 外付けI2Cハブ
              │  (M5 PaHub) │
              └──────┬──────┘
                     │
       ┌─────┬──────┼──────┬─────┐
       │     │      │      │     │
      ch0   ch1    ch2    ch3   ch4...
       │     │      │      │
    SHT40  SCD41  BMP280  P82B96
   (0x44) (0x62)  (0x76)    │
                            │
                     長距離ケーブル(5-20m)
                            │
                        遠隔センサー
```

---

## References

- [P82B96 Datasheet (TI)](https://www.ti.com/product/P82B96)
- [P82B96 Datasheet (NXP)](https://www.nxp.com/docs/en/data-sheet/P82B96.pdf)
- [TCA9548A Datasheet (TI)](https://www.ti.com/product/TCA9548A)
- [JLCPCB Capabilities](https://jlcpcb.com/capabilities/pcb-capabilities)
- [LCSC Electronics](https://www.lcsc.com/)
- [KiCad JLCPCB Plugin](https://github.com/Bouni/kicad-jlcpcb-tools)
- [M5Stack PaHub2](https://docs.m5stack.com/en/unit/pahub2)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-05 | Initial release |
| 1.1.0 | 2026-02-06 | I2Cハブ構成B、5Vファン制御、ECセンサーノイズ対策追加 |

---

**Skill Author**: Arsprout Analysis Team
**License**: MIT
