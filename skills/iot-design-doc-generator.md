# IoT Design Document Generator - Skill Definition

> 統合済み: iot-system-spec-generator の内容を含む

**Skill ID**: `iot-design-doc-generator`
**Category**: Documentation / IoT
**Version**: 1.0.0
**Created**: 2026-02-06
**Invocation**: `/iot-design-doc-generator` or `/iot-doc`

---

## Overview

This skill generates standardized design documents for IoT sensor/actuator nodes. It follows an 8-section structure covering all aspects from hardware to Home Assistant integration.

**Use Cases:**
- New IoT node design documentation
- Standardizing existing project documentation
- Rapid prototyping documentation
- Knowledge transfer and handoff

---

## Input Parameters

When invoking this skill, provide the following information:

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `project_name` | Project name (Japanese/English) | ミニ気象ステーション |
| `project_id` | Short identifier | mini-weather-station |
| `sensors` | List of sensors with models | SCD41, VEML6075 |
| `board` | Main controller board | W5500-EVB-Pico-PoE |

### Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `actuators` | List of actuators | None |
| `protocol` | Communication protocol | MQTT |
| `house_id` | Greenhouse/house identifier | h1 |
| `language` | Output language | ja |
| `power_source` | Power method | PoE |

---

## Output Format (8 Sections)

The generated document follows this structure:

### Section 1: 概要・目的 (Overview)

```markdown
## 1. 概要・目的

### ユースケース
- [Primary use case]
- [Secondary use case]

### 測定項目
| 項目 | センサー | 用途 |
|------|---------|------|
| {item} | {sensor} | {purpose} |
```

### Section 2: システム構成図 (Architecture)

```markdown
## 2. システム構成図

### 全体構成
```
┌─────────────────────────────────────────┐
│  {project_name}                          │
│  ┌──────────────┐    ┌──────────────┐   │
│  │  {sensor_1}  │──▶ │  {board}     │   │
│  │  {sensor_2}  │    │              │   │
│  └──────────────┘    └──────┬───────┘   │
└─────────────────────────────┼───────────┘
                              │ {protocol}
                              ▼
                    ┌─────────────────┐
                    │  MQTT Broker    │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  Home Assistant │
                    └─────────────────┘
```

### データフロー
[Sensor → MCU → MQTT → HA flow diagram]
```

### Section 3: ハードウェア (Hardware)

```markdown
## 3. ハードウェア

### 部品リスト
| カテゴリ | 品名 | 型番 | 入手先 | 単価 | 備考 |
|---------|------|------|--------|------|------|
| メインボード | {board} | {model} | {source} | ¥{price} | {notes} |
| センサー | {sensor} | {model} | {source} | ¥{price} | {notes} |

**概算コスト**: ¥{total}

### 配線・接続
```
{board}
├── I2C0 SDA（GP4）
│   ├── {sensor_1}（{i2c_addr_1}）
│   └── {sensor_2}（{i2c_addr_2}）
├── I2C0 SCL（GP5）
└── Ethernet PoE
```
```

### Section 4: ソフトウェア (Software)

```markdown
## 4. ソフトウェア

### ファームウェア概要（CircuitPython）

#### 主要ライブラリ
- `adafruit_wiznet5k`: Ethernet通信
- `adafruit_minimqtt`: MQTTクライアント
- `adafruit_{sensor_lib}`: センサードライバ

#### 処理フロー
```python
# メインループ疑似コード
while True:
    # センサー読み取り
    data = read_sensors()

    # MQTT送信
    mqtt_client.publish(TOPIC_SENSOR, json.dumps(data))

    # 間隔待機
    time.sleep(INTERVAL)
```

### サンプルコード
[Full code example with comments]
```

### Section 5: MQTTトピック設計 (MQTT Topics)

```markdown
## 5. MQTTトピック設計

### トピック構造
```
greenhouse/
├── {house_id}/
│   ├── sensors/
│   │   └── {sensor_type}     # センサーデータ
│   ├── actuators/
│   │   └── {actuator_type}/
│   │       ├── set           # 制御コマンド
│   │       └── state         # 状態フィードバック
│   └── node/
│       └── {node_id}/
│           └── status        # ノード状態
```

### ペイロード例
```json
{
  "{measurement_1}": {value},
  "{measurement_2}": {value},
  "timestamp": "{iso_timestamp}"
}
```
```

### Section 6: Home Assistant連携 (HA Integration)

```markdown
## 6. Home Assistant連携

### MQTT Discovery
```python
discovery_config = {
    "name": "{friendly_name}",
    "state_topic": "{topic}",
    "value_template": "{{ value_json.{field} }}",
    "unit_of_measurement": "{unit}",
    "device_class": "{class}",
    "unique_id": "{unique_id}",
    "device": {
        "identifiers": ["{device_id}"],
        "name": "{device_name}",
        "manufacturer": "DIY"
    }
}
```

### 自動化例
```yaml
automation:
  - alias: "{automation_name}"
    trigger:
      - platform: numeric_state
        entity_id: sensor.{entity}
        above: {threshold}
    action:
      - service: notify.{notifier}
        data:
          message: "{alert_message}"
```

### ダッシュボードカード
```yaml
type: entities
title: {project_name}
entities:
  - entity: sensor.{entity_1}
  - entity: sensor.{entity_2}
```
```

### Section 7: 実装手順 (Implementation)

```markdown
## 7. 実装手順

### Phase 1: ハードウェア組立
1. {board}の準備
2. センサー接続
3. 筐体組み込み（必要な場合）

### Phase 2: ファームウェアセットアップ
1. CircuitPython書き込み
2. ライブラリコピー
3. code.py配置・設定

### Phase 3: MQTT設定
1. ブローカー接続確認
2. トピック動作確認

### Phase 4: Home Assistant連携
1. MQTT Discovery確認
2. エンティティ命名
3. ダッシュボード追加

### Phase 5: 動作確認
1. センサー値確認
2. 通知テスト
3. 長期動作確認
```

### Section 8: 将来拡張案 (Future Enhancements)

```markdown
## 8. 将来拡張案

### {enhancement_1}
[Description and implementation approach]

### {enhancement_2}
[Description and implementation approach]

### 参考リンク
- [{reference_1}]({url_1})
- [{reference_2}]({url_2})
```

---

## Template Generation Rules

### Sensor-Specific Templates

| Sensor Type | Default Library | I2C Address | Measurements |
|-------------|-----------------|-------------|--------------|
| SCD41 | adafruit_scd4x | 0x62 | CO2, temp, humidity |
| SHT40 | adafruit_sht4x | 0x44 | temp, humidity |
| BMP280 | adafruit_bmp280 | 0x76/0x77 | pressure, temp |
| VEML6075 | adafruit_veml6075 | 0x10 | UV index |
| BH1750 | adafruit_bh1750 | 0x23 | lux |

### Board-Specific Templates

| Board | Firmware | Network | Power | Features |
|-------|----------|---------|-------|----------|
| W5500-EVB-Pico-PoE | CircuitPython | Ethernet | PoE | Industrial |
| Pico 2 W | CircuitPython | WiFi | USB/Battery | Compact |
| ESP32-CAM | Arduino | WiFi | USB | Camera |

### MQTT Topic Patterns

```
# Sensor data
greenhouse/{house}/sensors/{sensor_type}

# Actuator control
greenhouse/{house}/actuators/{actuator_type}/set
greenhouse/{house}/actuators/{actuator_type}/state

# Node status
greenhouse/{house}/node/{node_id}/status
```

---

## Usage Examples

### Example 1: Basic Sensor Node

```
User: /iot-design-doc-generator
      プロジェクト: 温湿度モニター
      センサー: SHT40
      ボード: W5500-EVB-Pico-PoE
```

**Output**: Complete design document with:
- Temperature/humidity monitoring overview
- Simple architecture diagram
- SHT40 + W5500-EVB-Pico-PoE hardware list
- CircuitPython firmware with adafruit_sht4x
- MQTT topics for temp/humidity
- HA sensor entities
- 5-phase implementation steps
- Future enhancements (alerts, logging)

### Example 2: Multi-Sensor Node

```
User: /iot-design-doc-generator
      プロジェクト: ハウス環境モニター
      センサー: SCD41, BH1750, VEML6075
      ボード: W5500-EVB-Pico-PoE
      ハウスID: h2
```

**Output**: Complete design document with:
- Multi-parameter monitoring overview
- Complex architecture with I2C hub consideration
- Multiple sensor hardware list with I2C addresses
- Firmware handling multiple sensors
- MQTT topics for CO2, temp, humidity, lux, UV
- HA dashboard with multiple cards
- I2C scan verification in implementation
- Future: threshold alerts, trend analysis

### Example 3: Actuator Node

```
User: /iot-design-doc-generator
      プロジェクト: 換気制御
      センサー: SHT40
      アクチュエータ: 換気扇リレー
      ボード: Pico 2 W
```

**Output**: Complete design document with:
- Ventilation control overview
- Sensor + actuator architecture
- Relay wiring and flyback diode
- Control logic (threshold-based)
- MQTT topics for both sensor and actuator
- HA switch entity + automation
- Safety considerations
- Future: PID control, scheduling

---

## Document Quality Checklist

The generated document must include:

- [ ] Clear project purpose and use cases
- [ ] ASCII architecture diagram
- [ ] Complete parts list with prices
- [ ] Wiring diagram
- [ ] Working code example
- [ ] MQTT topic structure
- [ ] HA configuration snippets
- [ ] Step-by-step implementation
- [ ] At least 2 future enhancement ideas
- [ ] References section

---

## Related Skills

- `sensor-driver-generator`: Generate sensor driver code
- `pico-wifi-mqtt-template`: WiFi MQTT firmware template
- `homeassistant-agri-starter`: HA integration patterns
- `agri-iot-board-design-template`: PCB design guide

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-06 | Initial release |

---

**Skill Author**: Arsprout Analysis Team
**License**: MIT

---

## 統合元からの補足: iot-system-spec-generator

### システム仕様書の10セクション構成（農業/工場/BEMS/スマートホーム対応）

iot-system-spec-generator は、IoTシステム全体の仕様書を10セクション構成で生成します。iot-design-doc-generator（8セクション）がノード単位の設計に対し、こちらはシステム全体の俯瞰的な仕様書です。

#### 10セクション構成

1. **システム概要**: プロジェクト名、目的、対象ユーザー、想定規模
2. **設計原則**: ローカル自律、通信断対応、OSS・セルフホスト等
3. **アーキテクチャ**: 3層構成（デバイス/GW/クラウド）、全体構成図
4. **ハードウェア構成**: デバイス一覧、センサー、アクチュエータ
5. **ソフトウェア構成**: GW層・デバイス層のソフトウェアスタック
6. **通信仕様**: MQTTトピック設計、QoS設定、通信頻度
7. **API層設計**: RESTエンドポイント、レスポンス形式
8. **ダッシュボード設計**: 規模別UI設計（小規模/中規模/大規模）
9. **ドメイン特有機能**: 農業/工場/BEMS/スマートホーム固有の機能
10. **今後の課題**: Phase 1/2/3の課題リスト

#### ドメイン別MQTTトピック構造

**農業（agriculture）**
```
farm/
├── sensors/
│   ├── {device_id}/
│   │   ├── temperature       # 温度 (℃)
│   │   ├── humidity          # 湿度 (%)
│   │   ├── co2               # CO2 (ppm)
│   │   ├── soil_moisture     # 土壌水分 (%)
│   │   └── radiation         # 日射 (W/m²)
├── actuators/
│   ├── {device_id}/
│   │   ├── valve/cmd         # バルブ制御
│   │   ├── fan/cmd           # ファン制御
│   │   └── state
└── alerts/
    ├── high_temp
    ├── low_humidity
    └── vpd_warning
```

**工場（factory）**
```
factory/
├── line/{line_id}/
│   ├── machine/{machine_id}/
│   │   ├── status            # 運転/停止/故障
│   │   ├── cycle_time        # サイクルタイム
│   │   ├── vibration         # 振動値
│   │   └── temperature
│   ├── production/
│   │   ├── count             # 生産数
│   │   ├── good              # 良品数
│   │   └── defect            # 不良数
└── alerts/
    ├── machine_fault
    ├── quality_warning
    └── maintenance_due
```

**ビル管理（bems）**
```
building/
├── floor/{floor_id}/
│   ├── zone/{zone_id}/
│   │   ├── temperature
│   │   ├── humidity
│   │   ├── co2
│   │   ├── occupancy         # 在室人数
│   │   └── lighting          # 照度
│   ├── hvac/
│   │   ├── setpoint          # 設定温度
│   │   ├── mode              # 冷房/暖房/送風
│   │   └── status
└── energy/
    ├── power                  # 電力 (kW)
    ├── demand                 # デマンド値
    └── cumulative             # 積算電力量
```

**スマートホーム（home）**
```
home/
├── room/{room_id}/
│   ├── temperature
│   ├── humidity
│   ├── motion                 # 人感
│   ├── door                   # 開閉
│   └── light/
│       ├── cmd
│       └── state
├── security/
│   ├── armed                  # 警戒モード
│   ├── alarm                  # アラーム状態
│   └── camera/{camera_id}
└── presence/
    ├── home                   # 在宅/外出
    └── occupants              # 在宅者
```

#### ドメイン特有機能の設計パターン

**農業（agriculture）**
- 8時間帯タイマー（変温管理）
- 日出/日入連動制御
- 飽差計算・アラート
- 灌水シーケンス制御
- 積算日射・積算温度

**工場（factory）**
- OEE（設備総合効率）計算
- 予知保全アラート
- 品質管理（SPC）
- 生産カウンター
- エネルギー監視

**ビル管理（bems）**
- デマンド制御
- 空調スケジュール
- 照明制御
- 電力監視
- 快適性指標（PMV/PPD）

**スマートホーム（home）**
- 在宅/外出モード
- シーン制御
- セキュリティ連携
- エネルギー監視
- 音声アシスタント連携

#### ダッシュボード規模別設計

| 規模 | 推奨技術 | 特徴 |
|------|---------|------|
| 小規模（1サイト） | Home Assistant | 簡単設定、モバイルアプリ完成度高 |
| 中規模（2〜10サイト） | Grafana | 柔軟なダッシュボード、SQLクエリ対応 |
| 大規模（10+サイト） | ThingsBoard | マルチテナント、デバイス管理機能 |

#### アーキテクチャ図テンプレート（農業用）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         農業IoTシステム アーキテクチャ                        │
└──────────────────────────────────────────────────────────────────────────────┘

                                   インターネット
                                        │
                                   [4G モデム]
                                        │
┌───────────────────────────────────────┼───────────────────────────────────────┐
│                           ハウス内ローカルネットワーク                         │
│                                       │                                       │
│   ┌───────────────────────────────────┴───────────────────────────────────┐   │
│   │                    Raspberry Pi (ローカルGW)                           │   │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│   │  │Home Asst │  │ Node-RED │  │SQLite    │  │Dashboard │              │   │
│   │  │(MQTT)    │  │          │  │          │  │          │              │   │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────┘              │   │
│   └───────────────────────────────────┬───────────────────────────────────┘   │
│                                       │                                       │
│              ┌────────────────────────┴────────────────────────┐              │
│              │                 MQTT (port 1883)                │              │
│              │                                                 │              │
│      ┌───────┴────────┐                               ┌────────┴───────┐      │
│  ┌───┴───┐        ┌───┴───┐                       ┌───┴───┐        ┌───┴───┐  │
│  │センサー│        │制御   │                       │センサー│        │制御   │  │
│  │ノード1 │        │ノード1│                       │ノード2 │        │ノード2│  │
│  └───┬───┘        └───┬───┘                       └───┬───┘        └───┬───┘  │
│      │                │                               │                │      │
│  [温湿度]          [電磁弁]                        [CO2]           [ファン]   │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

#### 入力パラメータYAML例（農業IoT）

```yaml
# プロジェクト基本情報
project:
  name: "OpenGreenhouse"
  version: "1.0.0"
  purpose: "施設園芸向け環境制御"
  domain: "agriculture"

# 規模
scale:
  target_users:
    - "個人農家"
    - "JA（農協）"
  expected_sites: "1〜100棟"
  devices_per_site: "5〜20台"

# プラットフォーム
platform:
  gateway: "Raspberry Pi"
  controller: "Home Assistant"
  protocol: "MQTT"
  database: "SQLite"
  cloud: "optional"

# ハードウェア構成
hardware:
  sensors:
    - name: "温湿度センサー"
      model: "SHT41"
      interface: "I2C"
      count: 2
    - name: "CO2センサー"
      model: "SCD41"
      interface: "I2C"
      count: 1
  actuators:
    - name: "電磁弁"
      channels: 4
      voltage: "DC12V"
    - name: "サーキュレータ"
      channels: 2
      voltage: "AC100V"
  controllers:
    - type: "WiFi"
      model: "Pico 2 WH"
      count: 2
    - type: "Ethernet"
      model: "W5500-EVB-Pico-PoE"
      count: 2

# 通信設計
communication:
  mqtt_broker: "Mosquitto"
  topic_prefix: "farm"
  qos:
    sensor: 0
    control: 1
    alert: 1
  intervals:
    temperature: "10s"
    co2: "30s"
    status: "60s"

# 設計原則（カスタマイズ可能）
principles:
  - "制御はローカル自律"
  - "通信断でも動作継続"
  - "OSS・セルフホスト"
  - "セキュア（VPN）"
```

#### 使い分けガイド

| 用途 | 使用スキル |
|------|-----------|
| 単一ノードの詳細設計 | iot-design-doc-generator（8セクション） |
| システム全体の仕様書 | iot-system-spec-generator（10セクション） |
| HA統合の詳細設計 | homeassistant-agri-starter |

この補足により、ノード単位の設計書（8セクション）とシステム全体の仕様書（10セクション）の両方に対応可能となります。
