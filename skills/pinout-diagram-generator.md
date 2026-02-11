# MCU/SBC ピン配置図ジェネレーター

MCU（マイクロコントローラ）やSBC（シングルボードコンピュータ）のピン配置図、配線図、GPIO割り当て表をASCIIアートとMarkdown形式で自動生成するスキル。

## 使用方法

```
/pinout-diagram-generator <ボード名> [オプション]
```

### パラメータ

| パラメータ | 説明 | 例 |
|-----------|------|-----|
| ボード名 | 対象ボード | `pico-w`, `pico2-w`, `esp32`, `w5500-evb-pico` |
| --sensors | 接続センサー（カンマ区切り） | `sht40,bmp280,scd41` |
| --actuators | アクチュエーター（カンマ区切り） | `relay4ch,solenoid` |
| --adc | ADC使用（カンマ区切り） | `pressure,ph,ec` |
| --i2c-bus | I2Cバスピン | `gp8,gp9` |
| --spi-bus | SPIバスピン | `gp16,gp17,gp18,gp19` |
| --output | 出力形式 | `full`, `table`, `ascii` |
| --lang | 言語 | `ja`, `en` |

## 対応ボード一覧

### Raspberry Pi Pico系

| ボードID | 名称 | 特徴 |
|----------|------|------|
| `pico` | Raspberry Pi Pico | RP2040, 26 GPIO |
| `pico-w` | Raspberry Pi Pico W | RP2040 + WiFi |
| `pico2` | Raspberry Pi Pico 2 | RP2350, 26 GPIO |
| `pico2-w` | Raspberry Pi Pico 2 W | RP2350 + WiFi |
| `w5500-evb-pico` | W5500-EVB-Pico | RP2040 + W5500 Ethernet |
| `w5500-evb-pico2` | W5500-EVB-Pico2 | RP2350 + W5500 Ethernet |
| `w5500-evb-pico-poe` | W5500-EVB-Pico-PoE | RP2350 + W5500 + PoE |

### ESP32系

| ボードID | 名称 | 特徴 |
|----------|------|------|
| `esp32` | ESP32 DevKit | 38ピン、WiFi/BT |
| `esp32-s3` | ESP32-S3 DevKit | USB OTG対応 |
| `esp32-c3` | ESP32-C3 | RISC-V、低消費電力 |

### Arduino系

| ボードID | 名称 | 特徴 |
|----------|------|------|
| `arduino-uno` | Arduino Uno | ATmega328P |
| `arduino-nano` | Arduino Nano | コンパクト |
| `arduino-mega` | Arduino Mega | 54 GPIO |

## 対応センサー/アクチュエーター

### I2Cセンサー

| センサーID | 名称 | I2Cアドレス | 測定項目 |
|-----------|------|-------------|----------|
| `sht40` | SHT40 | 0x44 | 温度、湿度 |
| `sht41` | SHT41 | 0x44 | 温度、湿度（高精度） |
| `bmp280` | BMP280 | 0x76/0x77 | 気圧、温度 |
| `bme280` | BME280 | 0x76/0x77 | 気圧、温度、湿度 |
| `scd41` | SCD41 | 0x62 | CO2、温度、湿度 |
| `bh1750` | BH1750 | 0x23/0x5C | 照度 |
| `ads1115` | ADS1115 | 0x48 | 外部ADC 4ch |
| `stemma-soil` | STEMMA Soil | 0x36 | 土壌水分 |

### ADCセンサー

| センサーID | 名称 | 出力 | 備考 |
|-----------|------|------|------|
| `pressure` | 水圧センサー | 0.5-2.5V | 0-1MPa |
| `ph` | pHセンサー | 0.5-2.5V | pH 0-14 |
| `ec` | ECセンサー | 0-2V | 0-20mS/cm |
| `soil-analog` | 土壌水分（アナログ） | 0-3.3V | キャリブレーション必要 |

### パルス/デジタルセンサー

| センサーID | 名称 | 接続 | 備考 |
|-----------|------|------|------|
| `flow-yfs201` | YF-S201流量計 | パルス | 1-30L/min |
| `water-level` | 水位センサー | デジタル | フロートスイッチ |

### アクチュエーター

| アクチュエーターID | 名称 | GPIO数 | 備考 |
|-------------------|------|--------|------|
| `relay4ch` | 4chリレーモジュール | 4 | 3.3V対応推奨 |
| `relay8ch` | 8chリレーモジュール | 8 | 3.3V対応推奨 |
| `solenoid-ac24v` | 電磁弁AC24V | - | リレー経由 |
| `solenoid-dc12v` | 電磁弁DC12V | - | リレー経由 |
| `motor-dc` | DCモーター | 2 | PWM対応 |

## 出力形式

### 1. GPIO割り当て表

```markdown
| GPIO | ピン# | 機能 | 接続先 | 備考 |
|------|-------|------|--------|------|
| GP2 | 4 | リレー1 | 電磁弁1 | OUT |
| GP3 | 5 | リレー2 | 電磁弁2 | OUT |
| GP8 | 11 | I2C SDA | センサーバス | - |
| GP9 | 12 | I2C SCL | センサーバス | - |
```

### 2. ASCIIアート配線図

```
┌─────────────────────────────────┐
│  Raspberry Pi Pico 2 W         │
│                                │
│  (1) GP0  ─┐    ┌─ VBUS (40)  │
│  (2) GP1  ─┤    ├─ VSYS (39)  │
│  (3) GND  ─┤    ├─ GND  (38)  │
│  (4) GP2 ─┼────┼→ リレー IN1  │
│  (5) GP3 ─┼────┼→ リレー IN2  │
│  ...      │    │              │
│  (36)3V3 ─┴────┴→ リレー VCC  │
└─────────────────────────────────┘
```

### 3. I2Cアドレス表

```markdown
| センサー | I2Cアドレス | 機能 |
|----------|-------------|------|
| SHT40 | 0x44 | 温度・湿度 |
| BMP280 | 0x76 | 気圧 |
| SCD41 | 0x62 | CO2 |
```

### 4. 電源ピン表

```markdown
| ピン | 用途 | 接続先 |
|------|------|--------|
| VBUS (40) | 5V入力 | USB電源 |
| 3V3 OUT (36) | 3.3V出力 | センサーVCC |
| GND | グランド | 共通GND |
```

### 5. 注意事項リスト

- 電圧レベル（3.3V/5V互換性）
- 電流制限（ピンあたり最大電流）
- 使用禁止ピン（WiFiチップ用等）
- プルアップ/プルダウン要件

## サンプル出力

### 入力例

```
/pinout-diagram-generator pico2-w --sensors sht40,bmp280 --actuators relay4ch --i2c-bus gp8,gp9
```

### 出力例

```markdown
# Pico 2 W ピン配置図

## ボード情報

| 項目 | 仕様 |
|------|------|
| ボード | Raspberry Pi Pico 2 W |
| MCU | RP2350 |
| 通信 | WiFi 2.4GHz |
| GPIO電圧 | 3.3V |

## GPIO割り当て表

| GPIO | ピン# | 機能 | 接続先 | I2Cアドレス | 備考 |
|------|-------|------|--------|-------------|------|
| GP2 | 4 | リレー1 | relay4ch IN1 | - | OUT |
| GP3 | 5 | リレー2 | relay4ch IN2 | - | OUT |
| GP4 | 6 | リレー3 | relay4ch IN3 | - | OUT |
| GP5 | 7 | リレー4 | relay4ch IN4 | - | OUT |
| GP8 | 11 | I2C0 SDA | センサーバス | - | - |
| GP9 | 12 | I2C0 SCL | センサーバス | - | - |
| GP25 | - | (WiFi) | - | - | 使用禁止 |
| LED | - | ステータス | オンボード | - | - |

## I2Cセンサー

| センサー | アドレス | 測定項目 | 電源 |
|----------|----------|----------|------|
| SHT40 | 0x44 | 温度、湿度 | 3.3V |
| BMP280 | 0x76 | 気圧、温度 | 3.3V |

## 配線図

        ┌────────────────────────────┐
        │   Raspberry Pi Pico 2 W   │
        │                           │
  3V3 ──┤1                        40├── VBUS
  GND ──┤3                        38├── GND
  GP2 ★─┤4  リレー1              36├── 3V3 OUT → センサー/リレーVCC
  GP3 ★─┤5  リレー2                │
  GP4 ★─┤6  リレー3                │
  GP5 ★─┤7  リレー4                │
  GND ──┤8                          │
        │                           │
  GP8 ●─┤11 I2C SDA                │
  GP9 ●─┤12 I2C SCL                │
        │                           │
        └────────────────────────────┘
  ★ = リレー出力  ● = I2Cバス

## I2C接続図

    Pico 2 W                    センサー群
    ┌─────────┐
    │         │     ┌───────────────┐
    │  GP8 ───┼────→│ SHT40 (0x44) │
    │  GP9 ───┼────→│ SDA/SCL      │
    │         │     └───────────────┘
    │         │            │
    │         │     ┌──────┴────────┐
    │ 3V3 OUT─┼────→│ BMP280 (0x76)│
    │  GND ───┼────→│ VCC/GND      │
    └─────────┘     └───────────────┘

## 電源ピン

| ピン | 用途 | 接続先 |
|------|------|--------|
| VBUS (40) | 5V入力 | USB電源 |
| 3V3 OUT (36) | 3.3V出力 | センサー/リレーVCC |
| GND (3,8,13,18,23,28,33,38) | グランド | 共通GND |

## 注意事項

| 項目 | 内容 |
|------|------|
| WiFiピン | GP25はWiFiチップ制御に使用。ユーザー使用禁止 |
| LED制御 | `machine.Pin("LED")` で制御 |
| リレー電圧 | 3.3V対応リレーモジュールを使用すること |
| I2Cプルアップ | センサー内蔵の場合は外部不要 |
| 最大電流 | GPIO 1ピンあたり最大12mA |
```

## W5500-EVB-Pico-PoE 特殊ピン

W5500搭載ボードでは以下のピンがEthernetで使用済み：

| GPIO | 用途 | 備考 |
|------|------|------|
| GP16 | SPI MISO | W5500 |
| GP17 | SPI CS | W5500 |
| GP18 | SPI SCK | W5500 |
| GP19 | SPI MOSI | W5500 |
| GP20 | RST | W5500 |
| GP21 | INT | W5500 |

**これらのピンは他用途に使用禁止**

## ESP32 特殊ピン

| GPIO | 用途 | 備考 |
|------|------|------|
| GPIO0 | ブートモード | 起動時Low=書き込みモード |
| GPIO2 | ブートモード | 内蔵LED |
| GPIO6-11 | 内蔵Flash | 使用禁止 |
| GPIO34-39 | 入力専用 | 出力不可 |

## 参考データシート

### Pico系

| ボード | URL |
|--------|-----|
| Pico W | https://datasheets.raspberrypi.com/picow/pico-w-datasheet.pdf |
| Pico 2 W | https://datasheets.raspberrypi.com/pico2/pico-2-w-datasheet.pdf |
| RP2350 | https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf |

### ESP32系

| ボード | URL |
|--------|-----|
| ESP32 | https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf |
| ESP32-S3 | https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf |

## 注意事項

- 生成されたピン配置は参考情報。実際の配線前にデータシートで確認すること
- I2Cアドレスの競合がないか確認すること
- 電源容量（特に3.3V出力）を確認すること
- 誘導性負荷（リレー、モーター）にはフライバックダイオードを使用すること
- AC100V/200V作業は電気工事士資格が必要な場合あり
