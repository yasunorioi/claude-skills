# CSV Safe Wrapper Generator

CSV安全読み書きラッパー関数を自動生成するスキル。エンコーディング自動検出・アトミック書き込み・BOMハンドリング対応。

- Skill ID: csv-safe-wrapper-generator
- Category: code-generation
- Version: 1.0.0
- Created: 2026-02-07
- Platform: Python 3.9+

## 概要

PythonでCSVファイルを安全に読み書きするためのラッパー関数を生成する。日本語環境で頻発するエンコーディング問題（Shift_JIS/CP932/UTF-8 BOM混在）の自動検出、書き込み時のアトミック操作（tempfile + os.replace）によるデータ損失防止、型変換・バリデーション付きの堅牢なCSV操作コードを提供。

## 使用方法

ユーザーが以下のいずれかを要求したら、このスキルを使用：

- CSVの読み書きで文字化けを防ぎたい
- CSVファイルの安全な書き込み処理を実装したい
- エンコーディング自動検出付きのCSV読み込みが欲しい
- 日本語CSVの読み書きラッパーを作成したい

## 入力パラメータ

| 項目 | 説明 | 例 | 必須 |
|------|------|-----|------|
| module_name | 生成モジュール名 | csv_utils | No (default: csv_safe_io) |
| target_encodings | 対応エンコーディング | ["utf-8", "cp932", "euc-jp"] | No (default: auto) |
| use_chardet | chardet/charset-normalizer使用 | true | No (default: true) |
| output_encoding | 書き込み時エンコーディング | "utf-8" | No (default: "utf-8") |
| include_dataclass | dataclass変換機能を含む | true | No (default: false) |
| atomic_write | アトミック書き込み | true | No (default: true) |

## 出力形式

生成するモジュール構成：

```
csv_safe_io.py
├── detect_encoding(file_path) -> str
│   └── chardet/charset-normalizerでエンコーディング検出
├── read_csv_safe(file_path, ...) -> list[dict]
│   ├── エンコーディング自動検出
│   ├── BOM除去
│   └── 型変換（オプション）
├── write_csv_safe(file_path, rows, ...) -> None
│   ├── アトミック書き込み（tempfile + os.replace）
│   ├── BOM付与（オプション）
│   └── 自動バックアップ（オプション）
├── read_csv_as_dataclass(file_path, cls) -> list[T]
│   └── dataclassへの自動マッピング（オプション）
└── CSVError (例外クラス)
    ├── EncodingDetectionError
    └── CSVValidationError
```

## 実装パターン

### エンコーディング自動検出

```python
from __future__ import annotations

import csv
import os
import tempfile
from pathlib import Path
from typing import Any


def detect_encoding(file_path: str | Path, sample_size: int = 65536) -> str:
    """ファイルのエンコーディングを自動検出する。

    Args:
        file_path: 対象ファイルパス
        sample_size: 検出に使用するバイト数

    Returns:
        検出されたエンコーディング名

    Raises:
        EncodingDetectionError: 検出に失敗した場合
    """
    file_path = Path(file_path)
    raw = file_path.read_bytes()[:sample_size]

    # BOM検出（最優先）
    if raw.startswith(b"\xef\xbb\xbf"):
        return "utf-8-sig"
    if raw.startswith((b"\xff\xfe", b"\xfe\xff")):
        return "utf-16"

    # charset-normalizer（chardetより高速・高精度）
    try:
        from charset_normalizer import from_bytes

        result = from_bytes(raw).best()
        if result is not None:
            encoding = str(result.encoding)
            # cp932/shift_jis の正規化
            if encoding.lower() in ("shift_jis", "shift-jis"):
                return "cp932"
            return encoding
    except ImportError:
        pass

    # chardet フォールバック
    try:
        import chardet

        detected = chardet.detect(raw)
        if detected["confidence"] > 0.7:
            encoding = detected["encoding"]
            if encoding and encoding.lower() in ("shift_jis", "shift-jis"):
                return "cp932"
            return encoding or "utf-8"
    except ImportError:
        pass

    # 最終フォールバック: UTF-8 → CP932 の順で試行
    for enc in ("utf-8", "cp932", "euc-jp", "iso-2022-jp"):
        try:
            raw.decode(enc)
            return enc
        except (UnicodeDecodeError, LookupError):
            continue

    raise EncodingDetectionError(f"Cannot detect encoding: {file_path}")
```

### 安全なCSV読み込み

```python
def read_csv_safe(
    file_path: str | Path,
    encoding: str | None = None,
    strip_bom: bool = True,
    fieldnames: list[str] | None = None,
    skip_empty_rows: bool = True,
) -> list[dict[str, str]]:
    """CSVファイルを安全に読み込む。

    Args:
        file_path: CSVファイルパス
        encoding: エンコーディング（Noneで自動検出）
        strip_bom: BOMを除去するか
        fieldnames: カラム名の指定（Noneで1行目をヘッダ使用）
        skip_empty_rows: 空行をスキップするか

    Returns:
        辞書のリスト（各行が1辞書）
    """
    file_path = Path(file_path)

    if encoding is None:
        encoding = detect_encoding(file_path)

    with open(file_path, "r", encoding=encoding, newline="") as f:
        content = f.read()

    # BOM除去
    if strip_bom and content.startswith("\ufeff"):
        content = content[1:]

    reader = csv.DictReader(
        content.splitlines(),
        fieldnames=fieldnames,
    )

    rows: list[dict[str, str]] = []
    for row in reader:
        if skip_empty_rows and all(v is None or v.strip() == "" for v in row.values()):
            continue
        # None値を空文字に正規化
        rows.append({k: (v or "").strip() for k, v in row.items()})

    return rows
```

### アトミック書き込み

```python
def write_csv_safe(
    file_path: str | Path,
    rows: list[dict[str, Any]],
    fieldnames: list[str] | None = None,
    encoding: str = "utf-8",
    write_bom: bool = False,
    backup: bool = False,
) -> None:
    """CSVファイルをアトミックに書き込む。

    tempfileに書いてからos.replaceで原子的に置換する。
    書き込み途中でクラッシュしても元ファイルは壊れない。

    Args:
        file_path: 出力先CSVファイルパス
        rows: 書き込むデータ（辞書のリスト）
        fieldnames: カラム名（Noneでrows[0]のキーを使用）
        encoding: 出力エンコーディング
        write_bom: UTF-8 BOMを付与するか
        backup: 既存ファイルの.bakバックアップを作成するか
    """
    file_path = Path(file_path)

    if not rows:
        return

    if fieldnames is None:
        fieldnames = list(rows[0].keys())

    # 同一ディレクトリにtempfile作成（os.replaceの要件）
    dir_path = file_path.parent
    dir_path.mkdir(parents=True, exist_ok=True)

    fd, tmp_path = tempfile.mkstemp(
        dir=str(dir_path),
        prefix=".csv_tmp_",
        suffix=".csv",
    )

    try:
        enc = "utf-8-sig" if (encoding == "utf-8" and write_bom) else encoding
        with os.fdopen(fd, "w", encoding=enc, newline="") as f:
            writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
            writer.writeheader()
            writer.writerows(rows)

        # バックアップ作成
        if backup and file_path.exists():
            backup_path = file_path.with_suffix(file_path.suffix + ".bak")
            os.replace(str(file_path), str(backup_path))

        # アトミック置換
        os.replace(tmp_path, str(file_path))

    except BaseException:
        # 失敗時はtempfile削除
        try:
            os.unlink(tmp_path)
        except OSError:
            pass
        raise
```

### 例外クラス

```python
class CSVError(Exception):
    """CSV操作の基底例外"""


class EncodingDetectionError(CSVError):
    """エンコーディング検出失敗"""


class CSVValidationError(CSVError):
    """CSVデータのバリデーションエラー"""
```

## サンプル出力

### 基本的な使い方

```python
from csv_safe_io import read_csv_safe, write_csv_safe, detect_encoding

# エンコーディング自動検出で読み込み
rows = read_csv_safe("data/farmers.csv")
# -> [{"name": "田中太郎", "area_ha": "12.5"}, ...]

# エンコーディング確認
enc = detect_encoding("data/legacy_export.csv")
# -> "cp932"

# UTF-8でアトミック書き込み（BOM付き、Excel互換）
write_csv_safe(
    "output/result.csv",
    rows,
    encoding="utf-8",
    write_bom=True,
    backup=True,
)
```

## 注意事項

### 必須要素

- charset-normalizer または chardet によるエンコーディング自動検出
- BOM（Byte Order Mark）の検出と除去/付与
- tempfile + os.replace によるアトミック書き込み
- 型ヒント完備、docstring付き

### 推奨要素

- cp932/Shift_JIS の正規化（日本語環境の互換性）
- 空行スキップ、None値の正規化
- バックアップ機能
- Excel互換のBOM付きUTF-8出力オプション

### 依存パッケージ

| パッケージ | 用途 | 必須 |
|-----------|------|------|
| charset-normalizer | エンコーディング検出（推奨） | No（フォールバックあり） |
| chardet | エンコーディング検出（代替） | No（フォールバックあり） |

両方未インストールでもUTF-8/CP932/EUC-JP/ISO-2022-JPの試行で動作する。
