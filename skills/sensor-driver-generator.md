# Sensor Driver Generator

I2Cセンサードライバ（CircuitPython/MicroPython対応）を自動生成するスキル。

## 概要

センサー仕様（I2Cアドレス、コマンド、データ形式）からCircuitPython/MicroPython用のI2Cセンサードライバを自動生成する。adafruit_bus_device を使った標準構造、CRC/チェックサム検証、propertyベースの直感的APIを実装したプロフェッショナル品質のドライバコードを作成。

## 使用方法

ユーザーが以下のいずれかを要求したら、このスキルを使用：

- I2Cセンサーのドライバを作成したい
- CircuitPython/MicroPython用のセンサーライブラリが必要
- センサーのデータシートからドライバを実装
- 既存センサードライバの移植（Arduino→CircuitPython等）

## 入力パラメータ

### センサー仕様

| 項目 | 説明 | 例 |
|------|------|------|
| センサー型番 | センサーの型番 | SHT31, SCD30, BME280 |
| I2Cアドレス | デフォルトアドレスと代替アドレス | 0x44 (0x45) |
| 測定項目 | センサーが測定する物理量 | 温度、湿度、CO2、気圧 |
| 精度 | 測定精度 | 温度±0.2℃、湿度±2%RH |
| 測定範囲 | 測定可能な範囲 | -40～125℃、0～100%RH |

### I2C通信仕様

| 項目 | 説明 | 例 |
|------|------|------|
| コマンド形式 | コマンドのバイト数 | 8bit, 16bit（MSBファースト） |
| 測定コマンド | 測定開始コマンド | 0x2C06（高繰り返し性測定） |
| リセットコマンド | ソフトリセットコマンド | 0x30A2 |
| ステータスコマンド | ステータス読み取り | 0xF32D |
| その他コマンド | ヒーター制御、設定変更等 | 0x306D（ヒーターON） |

### データ形式

| 項目 | 説明 | 例 |
|------|------|------|
| データ長 | 読み取りデータのバイト数 | 6バイト |
| データ構造 | データ配置 | 温度2B + CRC1B + 湿度2B + CRC1B |
| エンディアン | バイトオーダー | ビッグエンディアン |
| 計算式 | 生値から物理量への変換式 | T = -45 + 175 * (raw / 65535) |
| CRC/チェックサム | 検証方式 | CRC-8（多項式: 0x31、初期値: 0xFF） |

### タイミング仕様

| 項目 | 説明 | 例 |
|------|------|------|
| 測定時間 | 測定完了までの時間 | 最大15.5ms |
| コマンド遅延 | コマンド送信後の待機時間 | 5ms |
| リセット後待機 | リセット後の待機時間 | 100ms |
| I2Cクロック | 推奨I2Cクロック周波数 | 50kHz（クロックストレッチング対応） |

## 出力形式

生成するドライバ構成：

```python
{sensor_model}.py
├── モジュールdocstring
│   ├── 概要
│   ├── Author/Date
│   └── 参考情報
├── import文
│   ├── time
│   ├── adafruit_bus_device.i2c_device
│   └── micropython (const使用時)
├── 定数定義
│   ├── I2Cアドレス
│   ├── コマンド定義（const使用）
│   └── タイミング定数
├── 例外クラス（必要時）
│   ├── SensorError (基底)
│   ├── CRCError
│   └── NotReadyError
├── メインクラス
│   ├── __init__(i2c, address=DEFAULT_ADDR)
│   ├── property: 測定値プロパティ（温度、湿度等）
│   ├── _read_xxx(): 内部読み取りメソッド
│   ├── _write_command(): コマンド送信
│   ├── _crc8() / _verify_crc(): チェックサム検証
│   ├── reset(): リセットメソッド
│   └── その他設定メソッド
└── ヘルパー関数（必要時）
    └── read_sensor(): 簡易測定関数
```

## 実装パターン

### 基本構造（adafruit_bus_device使用）

```python
from adafruit_bus_device.i2c_device import I2CDevice
from micropython import const

_SENSOR_DEFAULT_ADDR = const(0x44)
_CMD_MEASURE = const(0x2C06)

class SensorDriver:
    def __init__(self, i2c, address=_SENSOR_DEFAULT_ADDR):
        self.i2c_device = I2CDevice(i2c, address)
        self._buffer = bytearray(6)
        self.reset()
```

### propertyベースAPI

```python
@property
def temperature(self) -> float:
    """温度を℃で返す"""
    raw = self._read_temp_raw()
    return -45.0 + 175.0 * raw / 65535.0

@property
def relative_humidity(self) -> float:
    """相対湿度を%で返す"""
    raw = self._read_humidity_raw()
    return 100.0 * raw / 65535.0
```

### CRC検証パターン

```python
@staticmethod
def _crc8(data):
    """CRC-8チェックサム計算"""
    crc = 0xFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            if crc & 0x80:
                crc = (crc << 1) ^ 0x31
            else:
                crc = crc << 1
            crc &= 0xFF
    return crc

def _verify_crc(self, data, expected_crc):
    """CRC検証"""
    if self._crc8(data) != expected_crc:
        raise RuntimeError("CRC check failed")
```

### コマンド送信パターン（16bitコマンド）

```python
def _write_command(self, command):
    """16bitコマンド送信"""
    cmd_buffer = bytearray([
        (command >> 8) & 0xFF,  # MSB
        command & 0xFF           # LSB
    ])
    with self.i2c_device as i2c:
        i2c.write(cmd_buffer)
```

### データ読み取りパターン

```python
def _read_measurement(self):
    """測定データ読み取り"""
    # 測定コマンド送信
    self._write_command(_CMD_MEASURE)
    time.sleep(0.016)  # 測定待機

    # データ読み取り（6バイト）
    with self.i2c_device as i2c:
        i2c.readinto(self._buffer)

    # CRC検証
    self._verify_crc(self._buffer[0:2], self._buffer[2])
    self._verify_crc(self._buffer[3:5], self._buffer[5])

    # 生値抽出
    temp_raw = (self._buffer[0] << 8) | self._buffer[1]
    humid_raw = (self._buffer[3] << 8) | self._buffer[4]

    return temp_raw, humid_raw
```

## サンプル出力

### SHT31（温湿度センサー）ドライバ

```python
"""
CircuitPython driver for Sensirion SHT31 temperature and humidity sensor.

Author: multi-agent-shogun
Created: 2026-02-05
Based on: Adafruit SHT31-D library
"""

import time
from adafruit_bus_device.i2c_device import I2CDevice
from micropython import const

_SHT31_DEFAULT_ADDR = const(0x44)
_SHT31_MEAS_HIGHREP = const(0x2C06)
_SHT31_SOFTRESET = const(0x30A2)

class SHT31:
    """Driver for SHT31 temperature and humidity sensor."""

    def __init__(self, i2c, address=_SHT31_DEFAULT_ADDR):
        self.i2c_device = I2CDevice(i2c, address)
        self._buffer = bytearray(6)
        self.reset()
        time.sleep(0.01)

    @property
    def temperature(self) -> float:
        """Temperature in °C"""
        temp_raw, _ = self._read_temp_humidity()
        return -45.0 + 175.0 * temp_raw / 65535.0

    @property
    def relative_humidity(self) -> float:
        """Relative humidity in %"""
        _, humid_raw = self._read_temp_humidity()
        return 100.0 * humid_raw / 65535.0

    def _read_temp_humidity(self):
        """Read raw temperature and humidity"""
        self._write_command(_SHT31_MEAS_HIGHREP)
        time.sleep(0.016)

        with self.i2c_device as i2c:
            i2c.readinto(self._buffer)

        # CRC verification
        if self._buffer[2] != self._crc8(self._buffer[0:2]):
            raise RuntimeError("Temperature CRC check failed")
        if self._buffer[5] != self._crc8(self._buffer[3:5]):
            raise RuntimeError("Humidity CRC check failed")

        temp_raw = (self._buffer[0] << 8) | self._buffer[1]
        humid_raw = (self._buffer[3] << 8) | self._buffer[4]

        return temp_raw, humid_raw

    def reset(self):
        """Soft reset"""
        self._write_command(_SHT31_SOFTRESET)
        time.sleep(0.002)

    def _write_command(self, command):
        """Write 16-bit command"""
        cmd_buffer = bytearray([
            (command >> 8) & 0xFF,
            command & 0xFF
        ])
        with self.i2c_device as i2c:
            i2c.write(cmd_buffer)

    @staticmethod
    def _crc8(data):
        """CRC-8 checksum"""
        crc = 0xFF
        for byte in data:
            crc ^= byte
            for _ in range(8):
                if crc & 0x80:
                    crc = (crc << 1) ^ 0x31
                else:
                    crc = crc << 1
                crc &= 0xFF
        return crc
```

## 生成時の注意事項

### 必須要素

- ✅ adafruit_bus_device を使用（I2CDevice）
- ✅ propertyデコレータで直感的API
- ✅ CRC/チェックサム検証実装
- ✅ リセット機能
- ✅ docstring完備
- ✅ エラーハンドリング（RuntimeError）
- ✅ 使用例をdocstringに記載

### 推奨要素

- ✅ const を使った定数定義（メモリ節約）
- ✅ バッファをbytearray で事前確保
- ✅ コンテキストマネージャー（with I2CDevice）使用
- ✅ タイムアウト処理（長時間測定時）
- ✅ read_wait() メソッド（データ待機）
- ✅ ヘルパー関数（簡易測定用）

### 避けるべき実装

- ❌ グローバル変数の多用
- ❌ print文のデバッグコード残留
- ❌ 例外の無視（try-except-pass）
- ❌ ハードコーディング（マジックナンバー）
- ❌ 不必要な依存ライブラリ

## 参考実装

- /home/yasu/arsprout_analysis/lib/sht31.py（温湿度センサー）
- /home/yasu/arsprout_analysis/lib/scd30.py（CO2センサー）
- Adafruit CircuitPython ライブラリ群

## 対応センサー例

| センサー型番 | 測定項目 | I2Cアドレス | 特徴 |
|------------|---------|-----------|------|
| SHT31 | 温度・湿度 | 0x44/0x45 | CRC-8検証、高精度 |
| SCD30 | CO2・温度・湿度 | 0x61 | NDIR方式、IEEE754 float |
| BME280 | 気圧・温度・湿度 | 0x76/0x77 | Bosch製、補正係数必要 |
| AHT20 | 温度・湿度 | 0x38 | 低コスト、CRC検証 |
| SGP30 | TVOC・eCO2 | 0x58 | MOX方式、自己較正 |
| SCD40/41 | CO2・温度・湿度 | 0x62 | 低消費電力、小型 |
