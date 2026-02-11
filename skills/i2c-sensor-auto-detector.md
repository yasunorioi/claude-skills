# I2C Sensor Auto-Detector

I2Cセンサー自動検出モジュール（CircuitPython/MicroPython対応）を生成するスキル。

## 概要

I2Cバス上のセンサーを自動スキャンし、I2Cアドレスとプローブベース型番判別により接続されたセンサーを自動識別するモジュールを生成する。I2Cアドレスマップ方式とプローブベース型番判別を組み合わせた堅牢な検出ロジック、フェイルセーフ設計、ログ出力・デバッグ支援を実装。

## 使用方法

ユーザーが以下のいずれかを要求したら、このスキルを使用：

- I2Cセンサーを自動検出したい
- 複数センサーの接続確認を自動化
- プラグアンドプレイ型のセンサー初期化
- センサー型番の自動識別（同一アドレスの異なるセンサー判別）

## 入力パラメータ

### サポート対象センサー

| センサー型番 | I2Cアドレス | 測定項目 | 判別方法 |
|------------|-----------|---------|---------|
| SHT31 | 0x44 | 温度・湿度 | ステータスレジスタ読み取り |
| SHT41 | 0x44 | 温度・湿度 | シリアル番号コマンド |
| SCD30 | 0x61 | CO2・温度・湿度 | 単一候補（自動判定） |
| SCD40 | 0x62 | CO2・温度・湿度 | シリアル番号形式 |
| SCD41 | 0x62 | CO2・温度・湿度 | デフォルトSCD40 |
| BMP280 | 0x76/0x77 | 気圧・温度 | 単一候補（自動判定） |

### 検出パラメータ

| パラメータ | 説明 | デフォルト |
|----------|------|-----------|
| verbose | ログ出力の有無 | True |
| retry_count | 通信失敗時のリトライ回数 | 3 |
| timeout_ms | 通信タイムアウト | 100ms |
| probe_delay_ms | プローブ間の待機時間 | 10ms |

## 出力形式

生成する検出モジュール構成：

```python
sensor_detector.py
├── モジュールdocstring
├── import文
├── センサーアドレスマップ（SENSOR_MAP）
│   └── {address: [候補センサーリスト]}
├── SensorInfo クラス
│   ├── __init__(model, address, name)
│   ├── _get_default_name(): センサー名取得
│   └── __repr__(): 文字列表現
├── SensorDetector クラス
│   ├── __init__(i2c)
│   ├── scan(verbose): I2Cバススキャン
│   ├── _probe_sensor(addr): センサー型番判別
│   ├── _identify_xxx(): 型番別判別メソッド
│   ├── get_sensor_by_type(type): 種類別検索
│   ├── has_sensor(model): 存在確認
│   └── print_summary(): サマリー表示
└── ヘルパー関数
    ├── scan_i2c_bus(): 簡易スキャン
    ├── is_sensor_supported(): サポート確認
    └── get_sensor_candidates(): 候補取得
```

## 実装パターン

### センサーアドレスマップ

```python
SENSOR_MAP = {
    0x44: ["SHT31", "SHT41"],  # 複数候補 → プローブ必要
    0x61: ["SCD30"],           # 単一候補 → 自動判定
    0x62: ["SCD40", "SCD41"],  # 複数候補 → プローブ必要
    0x76: ["BMP280"],          # 単一候補
    0x77: ["BMP280"],          # 単一候補（代替アドレス）
}
```

### SensorInfo データクラス

```python
class SensorInfo:
    """検出されたセンサー情報"""

    def __init__(self, model, address, name=None):
        self.model = model        # "SHT31"
        self.address = address    # 0x44
        self.name = name or self._get_default_name(model)

    def _get_default_name(self, model):
        """デフォルトセンサー名を取得"""
        name_map = {
            "SHT31": "Temperature/Humidity",
            "SCD30": "CO2",
            "BMP280": "Pressure",
        }
        return name_map.get(model, "Unknown Sensor")

    def __repr__(self):
        return (f"SensorInfo(model={self.model}, "
                f"address=0x{self.address:02X}, "
                f"name={self.name})")
```

### メインスキャンロジック

```python
class SensorDetector:
    def __init__(self, i2c):
        self.i2c = i2c
        self.detected_sensors = []

    def scan(self, verbose=True):
        """I2Cバススキャン＋型番判別"""
        self.detected_sensors = []

        # I2Cバスロック
        while not self.i2c.try_lock():
            pass

        try:
            # アドレススキャン
            addresses = self.i2c.scan()

            if verbose:
                print(f"Found {len(addresses)} I2C device(s)")

            # 各デバイスを判別
            for addr in addresses:
                if addr in SENSOR_MAP:
                    sensor = self._probe_sensor(addr, verbose)
                    if sensor:
                        self.detected_sensors.append(sensor)
                else:
                    if verbose:
                        print(f"Unknown device: 0x{addr:02X}")

        finally:
            self.i2c.unlock()

        return self.detected_sensors
```

### プローブベース型番判別

```python
def _probe_sensor(self, addr, verbose=True):
    """センサー型番を判別"""
    candidates = SENSOR_MAP[addr]

    # 単一候補 → 自動判定
    if len(candidates) == 1:
        model = candidates[0]
        return SensorInfo(model, addr)

    # 複数候補 → プローブして判別
    if addr == 0x44:
        model = self._identify_sht_sensor(addr, verbose)
    elif addr == 0x62:
        model = self._identify_scd_sensor(addr, verbose)
    else:
        # デフォルト: 最初の候補
        model = candidates[0]

    return SensorInfo(model, addr) if model else None
```

### SHT31 vs SHT41 判別

```python
def _identify_sht_sensor(self, addr, verbose=True):
    """SHT31とSHT41を判別"""
    try:
        # SHT41: シリアル番号読み取り (0x89)
        buffer = bytearray(6)
        self.i2c.writeto(addr, bytes([0x89]))
        time.sleep(0.01)
        self.i2c.readfrom_into(addr, buffer)

        # データ受信成功 → SHT41
        if buffer[0] != 0x00 or buffer[1] != 0x00:
            return "SHT41"
    except Exception:
        pass

    try:
        # SHT31: ステータスレジスタ読み取り (0xF32D)
        buffer = bytearray(3)
        self.i2c.writeto(addr, bytes([0xF3, 0x2D]))
        time.sleep(0.01)
        self.i2c.readfrom_into(addr, buffer)

        # データ受信成功 → SHT31
        if buffer[0] != 0x00 or buffer[1] != 0x00:
            return "SHT31"
    except Exception:
        pass

    # デフォルト: SHT31
    return "SHT31"
```

### SCD40 vs SCD41 判別

```python
def _identify_scd_sensor(self, addr, verbose=True):
    """SCD40とSCD41を判別"""
    try:
        # シリアル番号読み取り (0x3682)
        # 両センサー共に応答するため、型番判別不可
        buffer = bytearray(9)
        self.i2c.writeto(addr, bytes([0x36, 0x82]))
        time.sleep(0.01)
        self.i2c.readfrom_into(addr, buffer)

        # デフォルト: SCD40（より一般的）
        if verbose:
            print(f"SCD40/SCD41 detected, defaulting to SCD40")
        return "SCD40"
    except Exception:
        pass

    return "SCD40"
```

### 型別センサー取得

```python
def get_sensor_by_type(self, sensor_type):
    """センサー種類で検索"""
    type_map = {
        "temperature": ["SHT31", "SHT41", "SCD30", "SCD40", "SCD41"],
        "humidity": ["SHT31", "SHT41", "SCD30", "SCD40", "SCD41"],
        "co2": ["SCD30", "SCD40", "SCD41"],
        "pressure": ["BMP280"],
    }

    if sensor_type not in type_map:
        return None

    models = type_map[sensor_type]

    for sensor in self.detected_sensors:
        if sensor.model in models:
            return sensor

    return None

def has_sensor(self, model):
    """特定型番の存在確認"""
    for sensor in self.detected_sensors:
        if sensor.model == model:
            return True
    return False
```

## サンプル出力

### 基本的な使用例

```python
import board
import busio
from sensor_detector import SensorDetector

# I2C初期化
i2c = busio.I2C(board.GP5, board.GP4)

# 検出器作成
detector = SensorDetector(i2c)

# スキャン実行
sensors = detector.scan(verbose=True)

# 結果表示
for sensor in sensors:
    print(f"Found: {sensor.model} at 0x{sensor.address:02X}")
```

### スキャン結果の例

```
==================================================
Starting I2C sensor detection...
==================================================
Found 2 I2C device(s)

[✓] Identified: SHT31 (Temperature/Humidity) at 0x44
[✓] Detected: SCD30 (CO2) at 0x61

==================================================
Detection complete: 2 sensor(s) identified
==================================================
```

### センサー種類別取得

```python
# CO2センサーを取得
co2_sensor = detector.get_sensor_by_type("co2")
if co2_sensor:
    print(f"CO2 sensor: {co2_sensor.model}")

# 温度センサーを取得
temp_sensor = detector.get_sensor_by_type("temperature")
if temp_sensor:
    print(f"Temperature sensor: {temp_sensor.model}")

# 特定型番の存在確認
if detector.has_sensor("SHT31"):
    print("SHT31 is connected")
```

## 生成時の注意事項

### 必須要素

- ✅ I2Cバスロック（try_lock/unlock）
- ✅ 例外処理（通信失敗時のフェイルセーフ）
- ✅ プローブ失敗時のデフォルト値
- ✅ verbose モードでのログ出力
- ✅ SensorInfo データクラス
- ✅ 型別・型番別検索機能

### 推奨要素

- ✅ リトライロジック（通信失敗時）
- ✅ タイムアウト処理
- ✅ サマリー表示機能（print_summary）
- ✅ ヘルパー関数（簡易スキャン等）
- ✅ docstring完備
- ✅ 使用例をdocstringに記載

### プローブ設計原則

1. **非破壊的**: プローブ操作でセンサー状態を変更しない
2. **フェイルセーフ**: 通信失敗時はデフォルト値を返す
3. **短時間**: プローブは10ms以内で完了
4. **確実性**: 誤判別を最小限に

### 型番判別の優先順位

```
1. ユニークコマンド応答（最優先）
   例: SHT41のシリアル番号コマンド (0x89)

2. ステータスレジスタ読み取り
   例: SHT31のステータスレジスタ (0xF32D)

3. データ形式の違い
   例: シリアル番号のフォーマット差異

4. デフォルト値（判別不可時）
   例: SCD40/SCD41 → SCD40をデフォルト
```

### 避けるべき実装

- ❌ 無限ループ（I2Cバスロック取得）
- ❌ 例外の無視（センサー判別失敗の隠蔽）
- ❌ 過度なプローブ（センサーへの負荷）
- ❌ ハードコーディング（アドレスマップ外部化推奨）
- ❌ グローバルI2Cオブジェクト

## エラーハンドリング

### 通信失敗時の挙動

```python
def _probe_sensor(self, addr, verbose=True):
    """プローブ（エラーハンドリング付き）"""
    try:
        candidates = SENSOR_MAP[addr]
        # プローブロジック
        ...
    except OSError as e:
        # I2C通信失敗
        if verbose:
            print(f"[!] I2C error at 0x{addr:02X}: {e}")
        return None
    except Exception as e:
        # その他のエラー
        if verbose:
            print(f"[!] Unexpected error at 0x{addr:02X}: {e}")
        return None
```

### デフォルト値の選択基準

- **普及度**: より一般的なセンサーをデフォルトに
- **下位互換性**: 新型より旧型をデフォルト（互換性重視）
- **安全性**: 誤動作のリスクが低い方を選択

## 拡張性

### センサーの追加方法

```python
# 1. SENSOR_MAP に追加
SENSOR_MAP = {
    0x44: ["SHT31", "SHT41"],
    0x38: ["AHT20"],  # 新規追加
    ...
}

# 2. SensorInfo の name_map に追加
def _get_default_name(self, model):
    name_map = {
        "SHT31": "Temperature/Humidity",
        "AHT20": "Temperature/Humidity",  # 新規追加
        ...
    }

# 3. 複数候補の場合、判別メソッド追加
def _identify_aht_sensor(self, addr, verbose=True):
    """AHT20 vs AHT21 判別"""
    ...
```

## 参考実装

- /home/yasu/arsprout_analysis/lib/sensor_detector.py（センサー自動検出）
- CircuitPython I2C バススキャン仕様
- Sensirion データシート（SHT31/SHT41/SCD30/SCD40/SCD41）

## 使用シーン

| シーン | 用途 |
|--------|------|
| プラグアンドプレイ | センサー接続後、自動的に型番判別 |
| デバッグ支援 | I2Cバス上の全デバイスを一覧表示 |
| 動的初期化 | 検出されたセンサーに応じて初期化 |
| 冗長構成 | 複数センサーから利用可能なものを選択 |
| 互換性対応 | SHT31/SHT41など互換センサーの自動切替 |
