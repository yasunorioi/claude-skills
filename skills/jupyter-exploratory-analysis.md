# Jupyter Exploratory Analysis - Skill Definition

**Skill ID**: `jupyter-exploratory-analysis`
**Category**: Data Analysis / Machine Learning
**Version**: 1.0.0
**Created**: 2026-02-18

---

## Overview

SQLiteからpandas経由でデータを読み込み、seabornによる相関ヒートマップと統計分析を行うJupyter Notebookを生成するパターン。`nbformat` でNotebookを構築し、`nbconvert` でコードを実行して出力（グラフ・統計値）を埋め込んだ状態のipynbファイルを生成する。

農業ハウス環境データ（温度・湿度・CO2・日射・露点）の探索的分析（EDA）への適用実績あり。

---

## Use Cases

- SQLiteに蓄積された時系列データの相関分析をNotebook形式でレポートにしたい
- seabornヒートマップ・散布図・箱ひげ図を組み合わせた探索的分析を自動生成したい
- 「コードを実行した結果（グラフ・数値）込み」の.ipynbファイルを生成したい
- 農業・IoT・センサーデータの変数間関係を可視化・定量化したい

### 適用実績

- unipi-agri-ha 農業ハウス環境分析（subtask_481）:
  - `analysis/integrated_data.db`（SQLite, 5,856行 × 18カラム）を入力
  - 変数間の相関ヒートマップ（全変数 / 夏季限定）
  - 露点マージン時系列・時間帯別分布分析
  - 主要発見: 露点マージン<3℃が全データの75.7%（深夜はほぼ結露発生中）
  - 出力: `analysis/01_correlation_dewpoint.ipynb`（実行済み, 2.1MB）

---

## Skill Input

1. **SQLiteファイルパス**: データが格納されたSQLiteデータベース
2. **テーブル名**: 分析対象のテーブル
3. **インデックスカラム**: 時系列インデックスとなるカラム名（例: `datetime_jst`）
4. **分析対象カラム群**: 相関分析・可視化を行うカラムのリスト
5. **出力Notebookパス**: 生成する `.ipynb` ファイルのパス

---

## Skill Output

1. **実行済みJupyter Notebook** (`.ipynb`): コード + 出力グラフ + 統計値が埋め込まれた状態
2. **図ファイル群** (`.png`): ヒートマップ・散布図・時系列プロットをfigures/ディレクトリに保存
3. **統計サマリ**: 欠損率・基本統計量・強相関ペア一覧

---

## Implementation Pattern

### Step 1: Notebook構造の定義

```python
import nbformat as nbf

def create_notebook() -> nbf.NotebookNode:
    """空のJupyter Notebookを作成"""
    nb = nbf.v4.new_notebook()
    nb["metadata"] = {
        "kernelspec": {
            "display_name": "Python 3",
            "language": "python",
            "name": "python3"
        },
        "language_info": {"name": "python", "version": "3.11.0"}
    }
    return nb


def add_markdown(nb: nbf.NotebookNode, text: str) -> None:
    nb.cells.append(nbf.v4.new_markdown_cell(text))


def add_code(nb: nbf.NotebookNode, code: str) -> None:
    nb.cells.append(nbf.v4.new_code_cell(code))
```

### Step 2: セットアップセルの追加

```python
SETUP_CODE = """
import sqlite3
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from pathlib import Path

# 表示設定
plt.rcParams['figure.figsize'] = (12, 8)
plt.rcParams['font.family'] = 'DejaVu Sans'
sns.set_style('whitegrid')
sns.set_palette('husl')

FIGURES_DIR = Path('figures')
FIGURES_DIR.mkdir(exist_ok=True)

print('Setup complete')
"""

def add_setup_cell(nb: nbf.NotebookNode) -> None:
    add_markdown(nb, "## 0. Setup")
    add_code(nb, SETUP_CODE)
```

### Step 3: データ読み込みセルの追加

```python
def add_data_loading_cell(
    nb: nbf.NotebookNode,
    db_path: str,
    table_name: str,
    index_col: str,
    target_columns: list[str],
) -> None:
    """SQLiteからpandasでデータを読み込むセルを追加"""
    add_markdown(nb, "## 1. データ読み込み")

    code = f"""
conn = sqlite3.connect("{db_path}")
df_raw = pd.read_sql(
    "SELECT * FROM {table_name}",
    conn,
    parse_dates=["{index_col}"]
)
conn.close()

df_raw["{index_col}"] = pd.to_datetime(df_raw["{index_col}"], utc=True).dt.tz_convert("Asia/Tokyo")
df = df_raw.set_index("{index_col}").sort_index()

# 対象カラムのみ選択
target_cols = {target_columns}
df_analysis = df[[c for c in target_cols if c in df.columns]].copy()

print(f"行数: {{len(df_analysis)}}")
print(f"期間: {{df_analysis.index.min()}} 〜 {{df_analysis.index.max()}}")
print()
print("欠損率:")
missing = (df_analysis.isnull().sum() / len(df_analysis) * 100).round(1)
print(missing.to_string())
"""
    add_code(nb, code)
```

### Step 4: 相関ヒートマップセルの追加

```python
def add_correlation_heatmap(
    nb: nbf.NotebookNode,
    filter_condition: str = "",
    title: str = "相関行列ヒートマップ",
    filename: str = "fig_correlation_heatmap.png",
) -> None:
    """seabornによる相関ヒートマップセルを追加"""
    add_markdown(nb, f"## 相関分析: {title}")

    code = f"""
# フィルタ条件（例: 夏季限定）
df_plot = df_analysis{filter_condition}.dropna()
corr = df_plot.corr()

# 下三角マスク
mask = np.triu(np.ones_like(corr, dtype=bool))

fig, ax = plt.subplots(figsize=(14, 12))
sns.heatmap(
    corr,
    mask=mask,
    annot=True,
    fmt='.2f',
    cmap='RdYlGn',
    center=0,
    vmin=-1, vmax=1,
    annot_kws={{'size': 7}},
    ax=ax,
)
ax.set_title('{title}', fontsize=14, pad=15)
plt.tight_layout()
plt.savefig(FIGURES_DIR / '{filename}', dpi=150, bbox_inches='tight')
plt.show()

# 強相関ペアを抽出（|r| > 0.7）
strong = []
for i in range(len(corr.columns)):
    for j in range(i+1, len(corr.columns)):
        r = corr.iloc[i, j]
        if abs(r) > 0.7:
            strong.append((corr.columns[i], corr.columns[j], round(r, 3)))

strong.sort(key=lambda x: abs(x[2]), reverse=True)
print(f"強相関ペア (|r|>0.7): {{len(strong)}}件")
for a, b, r in strong[:10]:
    print(f"  {{a}} ↔ {{b}}: {{r}}")
"""
    add_code(nb, code)
```

### Step 5: 時系列・分布分析セルの追加

```python
def add_timeseries_analysis(
    nb: nbf.NotebookNode,
    target_col: str,
    threshold: float,
    unit: str = "",
    filename: str = "fig_timeseries.png",
) -> None:
    """時系列プロット + 時間帯別分布分析セルを追加"""
    add_markdown(nb, f"## {target_col} 時系列・時間帯分布分析")

    code = f"""
series = df_analysis["{target_col}"].dropna()
threshold = {threshold}

# 図1: 時系列
fig, ax = plt.subplots(figsize=(14, 4))
ax.plot(series.index, series.values, alpha=0.5, linewidth=0.5)
ax.axhline(threshold, color='red', linestyle='--', label=f'閾値: {{threshold}}{unit}')
ax.set_ylabel('{target_col} ({unit})')
ax.set_title('{target_col} 時系列')
ax.legend()
plt.tight_layout()
plt.savefig(FIGURES_DIR / '{filename}', dpi=150, bbox_inches='tight')
plt.show()

# 統計サマリ
print(f"基本統計:")
print(series.describe().round(3).to_string())
print()

# 閾値以下の割合
below = (series < threshold).sum()
print(f"閾値{threshold}{unit}未満: {{below}} / {{len(series)}} = {{below/len(series)*100:.1f}}%")

# 時間帯別平均
hourly = series.groupby(series.index.hour).mean()
print(f"\\n時間帯別平均 (上位危険時間帯):")
print(hourly.sort_values().head(5).round(3).to_string())
"""
    add_code(nb, code)
```

### Step 6: Notebookの実行と保存

```python
import nbformat
from nbconvert.preprocessors import ExecutePreprocessor
import subprocess
import sys

def execute_and_save(nb: nbf.NotebookNode, output_path: str, timeout: int = 600) -> None:
    """
    NotebookをJupyterカーネルで実行し、出力込みで保存。
    timeout: 実行タイムアウト（秒）
    """
    ep = ExecutePreprocessor(timeout=timeout, kernel_name="python3")

    # 実行ディレクトリはNotebookと同じ場所に設定（相対パス解決のため）
    import os
    notebook_dir = os.path.dirname(os.path.abspath(output_path))

    try:
        ep.preprocess(nb, {"metadata": {"path": notebook_dir}})
        print("Notebook execution: SUCCESS")
    except Exception as e:
        print(f"Notebook execution WARNING: {e}")
        print("（エラーがあってもNotebookは保存します）")

    with open(output_path, "w", encoding="utf-8") as f:
        nbformat.write(nb, f)

    print(f"Notebook saved: {output_path}")


def build_and_run(
    db_path: str,
    table_name: str,
    index_col: str,
    target_columns: list[str],
    output_path: str,
) -> None:
    """メインエントリポイント: Notebookを構築して実行"""
    nb = create_notebook()
    add_setup_cell(nb)
    add_data_loading_cell(nb, db_path, table_name, index_col, target_columns)

    # 全変数相関ヒートマップ
    add_correlation_heatmap(nb, title="全変数相関行列", filename="fig1_correlation_all.png")

    # 夏季限定（6-9月）
    add_correlation_heatmap(
        nb,
        filter_condition='[(df_analysis.index.month >= 6) & (df_analysis.index.month <= 9)]',
        title="夏季限定（6-9月）相関行列",
        filename="fig2_correlation_summer.png",
    )

    execute_and_save(nb, output_path)
```

---

## Notes / Limitations

- **nbconvert実行環境**: `jupyter`, `nbformat`, `nbconvert`, `seaborn`, `matplotlib` が必要。`pip install jupyter nbconvert nbformat seaborn pandas` で準備
- **実行時間**: データ量やセル数に応じて変わるが、5,000行規模で1〜3分程度
- **日本語フォント**: サーバー環境ではIPAexフォントが未インストールのことが多い。`DejaVu Sans` (デフォルト) で代替可能（日本語は文字化けする場合あり。英語ラベル推奨）
- **タイムゾーン**: `pd.to_datetime(..., utc=True).dt.tz_convert("Asia/Tokyo")` でJST変換を忘れずに。tz-naiveとtz-awareの混在はエラーになる
- **メモリ**: 全データを一度に読み込むため、数十万行規模ではchunk読み込みを検討
- **Notebookの再実行**: 既存の `.ipynb` を再実行する場合は `ExecutePreprocessor` を直接読み込んで使う

---

**Skill Author**: 足軽2号提案（subtask_481）
**Last Updated**: 2026-02-18
