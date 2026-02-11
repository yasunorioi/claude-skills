# env-derived-values-calculator

温度・湿度から飽差/露点/絶対湿度を計算するモジュールを生成するスキル。

## 概要

農業環境制御における重要な派生値（飽差、露点、絶対湿度、VPD）を計算するPythonモジュールを生成する。
CircuitPython/MicroPython互換（mathモジュールのみ使用）。

## 使用方法

```
/env-derived-values-calculator [言語] [出力形式]
```

### 例

```
/env-derived-values-calculator python module
/env-derived-values-calculator javascript functions
```

## 入力パラメータ

| パラメータ | 必須 | 説明 | デフォルト |
|-----------|------|------|-----------|
| 言語 | No | python, javascript, c | python |
| 出力形式 | No | module, functions, class | module |

## 出力形式

Pythonモジュール `calculations.py` を生成

## 計算式

### 1. 飽和水蒸気圧 (Tetens式)

```
es = 6.1078 × 10^(7.5T / (T + 237.3))
```
- es: 飽和水蒸気圧 [hPa]
- T: 温度 [°C]

### 2. 飽和水蒸気量

```
SVD = 217 × es / (T + 273.15)
```
- SVD: 飽和水蒸気量 [g/m³]

### 3. 絶対湿度

```
AH = SVD × (RH / 100)
```
- AH: 絶対湿度 [g/m³]
- RH: 相対湿度 [%]

### 4. 飽差

```
HD = SVD - AH = SVD × (1 - RH/100)
```
- HD: 飽差 [g/m³]

### 5. 露点 (Magnus式)

```
α = (17.27 × T) / (237.7 + T) + ln(RH / 100)
DP = (237.7 × α) / (17.27 - α)
```
- DP: 露点温度 [°C]

## サンプル出力

### calculations.py

```python
"""
飽差・露点・絶対湿度 計算モジュール

温度と湿度から派生値を計算する純粋な計算関数群。
CircuitPython/MicroPython互換（mathモジュールのみ使用）。

UECS CCMタイプ:
  - InAirHD: 室内飽差 [g/m³]
  - InAirDP: 室内露点 [°C]
  - InAirAbsHumid: 室内絶対湿度 [g/m³]
"""

import math


def saturation_vapor_pressure(temp_c: float) -> float:
    """
    飽和水蒸気圧を計算 (Tetens式)

    Args:
        temp_c: 気温 [°C]

    Returns:
        飽和水蒸気圧 [hPa]

    検証値:
        0°C  -> 6.11 hPa
        20°C -> 23.37 hPa
        25°C -> 31.67 hPa
        30°C -> 42.43 hPa
    """
    return 6.1078 * (10.0 ** ((7.5 * temp_c) / (temp_c + 237.3)))


def saturation_vapor_density(temp_c: float) -> float:
    """
    飽和水蒸気量（飽和絶対湿度）を計算

    Args:
        temp_c: 気温 [°C]

    Returns:
        飽和水蒸気量 [g/m³]

    検証値:
        0°C  -> 4.85 g/m³
        20°C -> 17.30 g/m³
        25°C -> 23.03 g/m³
        30°C -> 30.38 g/m³
    """
    es = saturation_vapor_pressure(temp_c)
    abs_temp = temp_c + 273.15
    return 217.0 * es / abs_temp


def absolute_humidity(temp_c: float, rh_percent: float) -> float:
    """
    絶対湿度を計算

    Args:
        temp_c: 気温 [°C]
        rh_percent: 相対湿度 [%]

    Returns:
        絶対湿度 [g/m³]

    検証値 (25°C):
        RH 40% -> 9.21 g/m³
        RH 60% -> 13.82 g/m³
        RH 80% -> 18.42 g/m³
    """
    svd = saturation_vapor_density(temp_c)
    return round(svd * (rh_percent / 100.0), 2)


def humidity_deficit(temp_c: float, rh_percent: float) -> float:
    """
    飽差を計算

    飽差 = 飽和水蒸気量 - 現在の水蒸気量
    植物の蒸散に影響する重要な指標。
    一般的に 3-7 g/m³ が最適とされる。

    Args:
        temp_c: 気温 [°C]
        rh_percent: 相対湿度 [%]

    Returns:
        飽差 [g/m³]

    検証値 (25°C):
        RH 40% -> 13.82 g/m³ (高すぎ: 気孔閉鎖)
        RH 60% -> 9.21 g/m³ (やや高め)
        RH 80% -> 4.61 g/m³ (適正範囲)
    """
    svd = saturation_vapor_density(temp_c)
    current_vd = svd * (rh_percent / 100.0)
    return round(svd - current_vd, 2)


def dew_point(temp_c: float, rh_percent: float) -> float:
    """
    露点温度を計算 (Magnus式)

    空気中の水蒸気が凝結し始める温度。
    結露防止の管理に重要。

    Args:
        temp_c: 気温 [°C]
        rh_percent: 相対湿度 [%]

    Returns:
        露点温度 [°C]

    検証値 (25°C):
        RH 40% -> 10.5°C
        RH 60% -> 16.7°C
        RH 80% -> 21.3°C
    """
    if rh_percent <= 0:
        return temp_c

    a = 17.27
    b = 237.7

    alpha = ((a * temp_c) / (b + temp_c)) + math.log(rh_percent / 100.0)
    dp = (b * alpha) / (a - alpha)
    return round(dp, 1)


def vpd(temp_c: float, rh_percent: float) -> float:
    """
    VPD (Vapor Pressure Deficit) を計算

    飽和水蒸気圧と実際の水蒸気圧の差。
    植物生理学でよく使用される。
    一般的に 0.4-1.6 kPa が適正とされる。

    Args:
        temp_c: 気温 [°C]
        rh_percent: 相対湿度 [%]

    Returns:
        VPD [kPa]
    """
    es = saturation_vapor_pressure(temp_c)
    e = es * (rh_percent / 100.0)
    return round((es - e) / 10.0, 2)  # hPa -> kPa


def calculate_all(temp_c: float, rh_percent: float) -> dict:
    """
    全ての派生値を一括計算

    Args:
        temp_c: 気温 [°C]
        rh_percent: 相対湿度 [%]

    Returns:
        dict: {
            'absolute_humidity': 絶対湿度 [g/m³],
            'humidity_deficit': 飽差 [g/m³],
            'dew_point': 露点 [°C],
            'vpd': VPD [kPa],
            'saturation_vapor_density': 飽和水蒸気量 [g/m³]
        }

    Example:
        >>> result = calculate_all(25.0, 60.0)
        >>> print(result['humidity_deficit'])
        9.21
    """
    svd = saturation_vapor_density(temp_c)
    abs_humid = round(svd * (rh_percent / 100.0), 2)
    hd = round(svd - abs_humid, 2)
    dp = dew_point(temp_c, rh_percent)
    vpd_val = vpd(temp_c, rh_percent)

    return {
        'absolute_humidity': abs_humid,
        'humidity_deficit': hd,
        'dew_point': dp,
        'vpd': vpd_val,
        'saturation_vapor_density': round(svd, 2)
    }


# UECS CCM用ヘルパー
def get_uecs_derived_values(temp_c: float, rh_percent: float) -> dict:
    """
    UECS CCM送信用の派生値を取得

    Returns:
        dict: {
            'InAirHD': 飽差 [g/m³],
            'InAirDP': 露点 [°C],
            'InAirAbsHumid': 絶対湿度 [g/m³]
        }
    """
    return {
        'InAirHD': humidity_deficit(temp_c, rh_percent),
        'InAirDP': dew_point(temp_c, rh_percent),
        'InAirAbsHumid': absolute_humidity(temp_c, rh_percent)
    }
```

## 農業での活用

### 飽差管理

| 飽差 [g/m³] | 状態 | 対応 |
|------------|------|------|
| < 3 | 低すぎ | 病害リスク、換気促進 |
| 3-7 | 適正 | 維持 |
| > 7 | 高すぎ | 気孔閉鎖、加湿必要 |

### VPD管理

| VPD [kPa] | 状態 | 対応 |
|-----------|------|------|
| < 0.4 | 低すぎ | カビリスク |
| 0.4-1.6 | 適正 | 維持 |
| > 1.6 | 高すぎ | 蒸散過多 |

### 露点管理

- **露点 > 表面温度**: 結露発生
- **結露防止**: 露点より2-3°C高い温度を維持

## 使用例

```python
# センサー読み取り後の派生値計算
from lib.calculations import calculate_all, get_uecs_derived_values

# SHT40から温湿度取得
temp, humid = sht40.measurements

# 派生値計算
derived = calculate_all(temp, humid)
print(f"飽差: {derived['humidity_deficit']} g/m³")
print(f"露点: {derived['dew_point']}°C")

# UECS CCM送信
uecs_values = get_uecs_derived_values(temp, humid)
for ccm_type, value in uecs_values.items():
    sender.send(ccm_type, value)
```

## 検証値テーブル

25°C基準での検証値：

| RH [%] | 絶対湿度 [g/m³] | 飽差 [g/m³] | 露点 [°C] | VPD [kPa] |
|--------|---------------|------------|----------|----------|
| 40 | 9.21 | 13.82 | 10.5 | 1.90 |
| 50 | 11.52 | 11.52 | 13.9 | 1.58 |
| 60 | 13.82 | 9.21 | 16.7 | 1.27 |
| 70 | 16.12 | 6.91 | 19.0 | 0.95 |
| 80 | 18.42 | 4.61 | 21.3 | 0.63 |
| 90 | 20.73 | 2.30 | 23.2 | 0.32 |

## 関連スキル

- circuitpython-toml-config: 設定ファイル読み込み
