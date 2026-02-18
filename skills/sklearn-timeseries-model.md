# Sklearn Timeseries Model - Skill Definition

**Skill ID**: `sklearn-timeseries-model`
**Category**: Machine Learning / Data Science
**Version**: 1.0.0
**Created**: 2026-02-18

---

## Overview

時系列データをtrain/val/testに**時間順に**分割し、特徴量エンジニアリング（ラグ特徴量・cyclicalエンコーディング・交互作用特徴量）→GridSearchCV→回帰＋二値分類の両面評価→joblib保存までのパイプラインをJupyter Notebookで完結させるパターン。

LinearRegression / RandomForest / GradientBoosting（GridSearchCV）の複数モデル比較、月別分割によるtrain/val/test設計、特徴量重要度分析を含む農業・IoT時系列データへの適用実績あり。

---

## Use Cases

- SQLiteの時系列データからscikit-learnモデルを構築してjoblibで保存したい
- 時間的リーク（data leakage）を防ぐ厳密なtrain/val/test分割が必要
- 複数モデルをGridSearchCVで自動チューニングして性能比較したい
- 回帰精度（MAE/RMSE/R²）と二値分類精度（Precision/Recall/F1）を両方評価したい
- 特徴量重要度・残差分析・時系列予測プロットをNotebookで一括生成したい

### 適用実績

- unipi-agri-ha 露点マージン予測モデル（subtask_483）:
  - GradientBoosting: Test F1=0.940、結露リスク二値分類
  - GridSearchCV Best: lr=0.05, n_estimators=100, max_depth=5
  - 最重要特徴量: `om_shortwave_radiation`（importance=0.745）
  - 夜間(0-5時)で系統的な過大予測 → -1℃補正を推奨

- unipi-agri-ha 内温予測モデル（subtask_484）:
  - LR/RF/GB比較 → RandomForest最優秀（Test R²=0.7451, MAE=2.111℃）
  - GBR GridSearchCV Best: lr=0.05, max_depth=3, n_estimators=200, subsample=0.8
  - 最重要特徴量: `shortwave_rolling_2h`（2時間移動平均日射）

---

## Skill Input

1. **SQLiteファイルパス**: `integrated_data.db` 等の統合済みSQLite DB
2. **テーブル名・インデックスカラム**: データ取得用
3. **目的変数**: 予測したいカラム名（例: `dew_margin`, `in_air_temp`）
4. **分割設定**: train/val/test の月範囲（例: train=4-7月, val=8月, test=9月）
5. **二値分類閾値（オプション）**: 目的変数を二値に変換する閾値（例: 3.0で危険ゾーン）
6. **出力先**: Notebookパス、モデル保存先ディレクトリ

---

## Skill Output

1. **実行済みJupyter Notebook**: モデル構築〜評価までのコード＋出力が埋め込まれた `.ipynb`
2. **pickleモデルファイル**: `models/xxx_model.pkl`（joblib形式）
3. **評価図群** (`figures/`):
   - 特徴量重要度バーチャート
   - 実測値 vs 予測値散布図
   - テスト期間の時系列残差プロット
   - 混同行列（二値分類時）
   - モデル比較バーチャート

---

## Implementation Pattern

### Step 1: データ読み込みと時間順分割

```python
import sqlite3
import pandas as pd
import numpy as np

def load_and_split(
    db_path: str,
    table_name: str,
    index_col: str,
    target_col: str,
    train_months: list[int],
    val_months: list[int],
    test_months: list[int],
) -> tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    SQLiteからデータを読み込み、月でtrain/val/testに分割。
    時系列リーク防止: 必ず時間順に分割すること（シャッフル禁止）。
    """
    conn = sqlite3.connect(db_path)
    df = pd.read_sql(f"SELECT * FROM {table_name}", conn, parse_dates=[index_col])
    conn.close()

    df[index_col] = pd.to_datetime(df[index_col], utc=True).dt.tz_convert("Asia/Tokyo")
    df = df.set_index(index_col).sort_index()
    df = df.dropna(subset=[target_col])  # 目的変数の欠損は除外

    train = df[df.index.month.isin(train_months)]
    val   = df[df.index.month.isin(val_months)]
    test  = df[df.index.month.isin(test_months)]

    print(f"Train: {len(train)}行 ({train.index.min().date()} 〜 {train.index.max().date()})")
    print(f"Val:   {len(val)}行 ({val.index.min().date()} 〜 {val.index.max().date()})")
    print(f"Test:  {len(test)}行 ({test.index.min().date()} 〜 {test.index.max().date()})")

    return train, val, test
```

### Step 2: 特徴量エンジニアリング

```python
def build_features(df: pd.DataFrame, feature_cols: list[str]) -> pd.DataFrame:
    """
    時系列モデル用の特徴量を生成する。
    1. ラグ特徴量（1時間前の値）
    2. 差分特徴量（前時刻からの変化量）
    3. cyclicalエンコーディング（時刻の周期性を sin/cos で表現）
    4. 移動平均（過去N時間の平均）
    5. 交互作用特徴量（重要な変数の積）
    """
    result = df[feature_cols].copy()

    # 1. ラグ特徴量
    for col in feature_cols:
        result[f"{col}_lag1h"] = result[col].shift(1)

    # 2. 差分特徴量
    for col in feature_cols:
        result[f"{col}_diff1h"] = result[col].diff(1)

    # 3. cyclicalエンコーディング（時刻）
    hour = df.index.hour
    result["hour_sin"] = np.sin(2 * np.pi * hour / 24)
    result["hour_cos"] = np.cos(2 * np.pi * hour / 24)

    # 4. 夜間フラグ（20:00〜06:00）
    result["is_night"] = ((hour >= 20) | (hour < 6)).astype(int)

    # 5. 移動平均（例: 日射の2時間移動平均）
    if "om_shortwave_radiation" in feature_cols:
        result["shortwave_rolling_2h"] = (
            result["om_shortwave_radiation"].rolling(window=2, min_periods=1).mean()
        )

    # 6. 交互作用特徴量（例: 外気温 × 湿度）
    if "jma_air_temp" in feature_cols and "jma_humidity" in feature_cols:
        result["temp_x_humidity"] = result["jma_air_temp"] * result["jma_humidity"]

    return result.dropna()


def prepare_xy(
    split: pd.DataFrame,
    feature_cols_extended: list[str],
    target_col: str,
) -> tuple[pd.DataFrame, pd.Series]:
    X = split[feature_cols_extended].dropna()
    y = split.loc[X.index, target_col]
    return X, y
```

### Step 3: 複数モデル構築とGridSearchCV

```python
from sklearn.linear_model import LinearRegression
from sklearn.ensemble import RandomForestRegressor, GradientBoostingRegressor
from sklearn.model_selection import GridSearchCV
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

def train_models(
    X_train: pd.DataFrame,
    y_train: pd.Series,
    X_val: pd.DataFrame,
    y_val: pd.Series,
) -> dict:
    """
    LR/RF/GB の3モデルを構築し、Val性能で比較。
    GB はGridSearchCVで最適化。
    """
    results = {}

    # LinearRegression（ベースライン）
    lr = LinearRegression()
    lr.fit(X_train, y_train)
    y_pred_val = lr.predict(X_val)
    results["LinearRegression"] = {
        "model": lr,
        "val_mae": mean_absolute_error(y_val, y_pred_val),
        "val_rmse": np.sqrt(mean_squared_error(y_val, y_pred_val)),
        "val_r2": r2_score(y_val, y_pred_val),
    }

    # RandomForest
    rf = RandomForestRegressor(n_estimators=100, random_state=42, n_jobs=-1)
    rf.fit(X_train, y_train)
    y_pred_val = rf.predict(X_val)
    results["RandomForest"] = {
        "model": rf,
        "val_mae": mean_absolute_error(y_val, y_pred_val),
        "val_rmse": np.sqrt(mean_squared_error(y_val, y_pred_val)),
        "val_r2": r2_score(y_val, y_pred_val),
    }

    # GradientBoosting + GridSearchCV
    param_grid = {
        "learning_rate": [0.01, 0.05, 0.1],
        "n_estimators": [100, 200],
        "max_depth": [3, 5],
        "subsample": [0.8, 1.0],
    }
    gbr = GradientBoostingRegressor(random_state=42)
    gs = GridSearchCV(gbr, param_grid, cv=3, scoring="neg_mean_absolute_error", n_jobs=-1)
    gs.fit(X_train, y_train)

    best_gbr = gs.best_estimator_
    y_pred_val = best_gbr.predict(X_val)
    results["GradientBoosting"] = {
        "model": best_gbr,
        "best_params": gs.best_params_,
        "val_mae": mean_absolute_error(y_val, y_pred_val),
        "val_rmse": np.sqrt(mean_squared_error(y_val, y_pred_val)),
        "val_r2": r2_score(y_val, y_pred_val),
    }

    # Val性能で最優秀モデルを選定
    best_name = min(results, key=lambda k: results[k]["val_mae"])
    print(f"\nBest model on Val: {best_name}")
    return results, best_name
```

### Step 4: テストセット評価 + 二値分類評価

```python
from sklearn.metrics import precision_score, recall_score, f1_score, confusion_matrix

def evaluate_on_test(
    model,
    X_test: pd.DataFrame,
    y_test: pd.Series,
    binary_threshold: float | None = None,
) -> dict:
    """
    回帰精度（MAE/RMSE/R²）と二値分類精度（Precision/Recall/F1）を評価。
    binary_threshold: これ未満を「危険ゾーン」（label=1）と定義
    """
    y_pred = model.predict(X_test)

    metrics = {
        "test_mae": mean_absolute_error(y_test, y_pred),
        "test_rmse": np.sqrt(mean_squared_error(y_test, y_pred)),
        "test_r2": r2_score(y_test, y_pred),
    }

    if binary_threshold is not None:
        y_true_bin = (y_test < binary_threshold).astype(int)
        y_pred_bin = (y_pred < binary_threshold).astype(int)
        metrics.update({
            "precision": precision_score(y_true_bin, y_pred_bin),
            "recall": recall_score(y_true_bin, y_pred_bin),
            "f1": f1_score(y_true_bin, y_pred_bin),
            "confusion_matrix": confusion_matrix(y_true_bin, y_pred_bin).tolist(),
        })

    print("Test Metrics:")
    for k, v in metrics.items():
        if k != "confusion_matrix":
            print(f"  {k}: {v:.4f}")

    return metrics, y_pred
```

### Step 5: 可視化（特徴量重要度・残差分析・モデル比較）

```python
import matplotlib.pyplot as plt
import seaborn as sns
from pathlib import Path

def plot_feature_importance(model, feature_names: list[str], output_dir: Path, prefix: str = "") -> None:
    """特徴量重要度バーチャート"""
    if not hasattr(model, "feature_importances_"):
        return

    fi = pd.Series(model.feature_importances_, index=feature_names).sort_values(ascending=True)
    top20 = fi.tail(20)

    fig, ax = plt.subplots(figsize=(8, 8))
    top20.plot(kind="barh", ax=ax)
    ax.set_title("Feature Importances (Top 20)")
    ax.set_xlabel("Importance")
    plt.tight_layout()
    plt.savefig(output_dir / f"{prefix}feature_importance.png", dpi=150)
    plt.show()


def plot_prediction_vs_actual(y_test: pd.Series, y_pred: np.ndarray, output_dir: Path, prefix: str = "") -> None:
    """実測値 vs 予測値散布図 + 時系列プロット"""
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))

    # 散布図
    axes[0].scatter(y_test, y_pred, alpha=0.3, s=5)
    lims = [min(y_test.min(), y_pred.min()), max(y_test.max(), y_pred.max())]
    axes[0].plot(lims, lims, 'r--', lw=1)
    axes[0].set_xlabel("Actual")
    axes[0].set_ylabel("Predicted")
    axes[0].set_title("Actual vs Predicted")

    # 時系列
    axes[1].plot(y_test.index, y_test.values, label="Actual", alpha=0.7)
    axes[1].plot(y_test.index, y_pred, label="Predicted", alpha=0.7)
    axes[1].legend()
    axes[1].set_title("Test Period Timeseries")

    plt.tight_layout()
    plt.savefig(output_dir / f"{prefix}prediction_vs_actual.png", dpi=150)
    plt.show()


def plot_model_comparison(results: dict, output_dir: Path) -> None:
    """全モデルのVal MAE比較バーチャート"""
    names = list(results.keys())
    maes = [results[n]["val_mae"] for n in names]

    fig, ax = plt.subplots(figsize=(8, 4))
    ax.bar(names, maes)
    ax.set_ylabel("Val MAE")
    ax.set_title("Model Comparison (Val MAE)")
    plt.tight_layout()
    plt.savefig(output_dir / "model_comparison.png", dpi=150)
    plt.show()
```

### Step 6: モデル保存

```python
import joblib

def save_model(model, model_name: str, output_dir: str) -> str:
    """joblibでモデルをpickle保存"""
    Path(output_dir).mkdir(parents=True, exist_ok=True)
    path = str(Path(output_dir) / f"{model_name}.pkl")
    joblib.dump(model, path)
    print(f"Model saved: {path}")
    return path


def load_model(path: str):
    """保存済みモデルの読み込み"""
    return joblib.load(path)
```

---

## Notes / Limitations

- **時系列分割の鉄則**: `train_test_split(shuffle=True)` は厳禁。将来データがtrainに混入してリーク発生。必ず月・日付で時間順に分割
- **GridSearchCVのcv=3**: 時系列ではkfoldではなくTimeSeriesSplitが本来正しいが、月別分割で既にリークを防いでいる場合は通常のcv=3で十分
- **夜間補正**: 回帰モデルは時間帯によって系統的バイアスが生じることがある（深夜に過大予測等）。残差の時間帯別分析を必ず行い、必要なら時間帯別補正を追加
- **特徴量の順序**: ラグ・差分特徴量を追加すると先頭数行がNaNになる。`dropna()`でこれらを除くが、train/val/testの境界でも同様に確認
- **joblib vs pickle**: `joblib.dump` はscikit-learnの大きなndarrayに最適化されているためpickleより高速・コンパクト
- **依存ライブラリ**: `scikit-learn`, `pandas`, `numpy`, `matplotlib`, `seaborn`, `joblib`, `nbformat`, `nbconvert`

---

**Skill Author**: 足軽2号提案（subtask_483） / 足軽3号補完（subtask_484）
**Last Updated**: 2026-02-18
