# Dataclass Model Generator

SQLiteテーブル定義やCSVカラムからPython dataclass/NamedTupleモデルを自動生成するスキル。

- Skill ID: dataclass-model-generator
- Category: code-generation
- Version: 1.0.0
- Created: 2026-02-07
- Platform: Python 3.10+

## 概要

SQLiteテーブルのスキーマ情報やCSVのカラム構造から、型ヒント付きPython dataclassまたはNamedTupleモデル定義を自動生成する。from_dict/to_dict変換メソッド、バリデーションプロパティ、SQLiteカラム型からPython型への自動マッピングを含む実用的なデータモデルコードを出力する。

## 使用方法

ユーザーが以下のいずれかを要求したら、このスキルを使用：

- SQLiteテーブルからデータモデルを生成したい
- CSVカラムからdataclassを作成したい
- NamedTupleモデルを自動生成したい
- データベースのORMライクなモデルクラスが欲しい

## 入力パラメータ

| 項目 | 説明 | 例 | 必須 |
|------|------|-----|------|
| source_type | ソース種別 | "sqlite" / "csv" / "manual" | Yes |
| db_path | SQLiteファイルパス | "data/app.db" | source=sqlite時 |
| table_name | テーブル名 | "users" | source=sqlite時 |
| csv_path | CSVファイルパス | "data/sample.csv" | source=csv時 |
| columns | カラム定義（手動） | [{"name": "id", "type": "int"}] | source=manual時 |
| model_type | 出力形式 | "dataclass" / "namedtuple" | No (default: dataclass) |
| include_validation | バリデーション含む | true | No (default: true) |
| include_conversion | 変換メソッド含む | true | No (default: true) |

### SQLite型→Python型マッピング

| SQLite型 | Python型 | 備考 |
|----------|---------|------|
| INTEGER | int | PRIMARY KEYの場合は `int \| None` |
| REAL | float | |
| TEXT | str | |
| BLOB | bytes | |
| BOOLEAN | bool | SQLiteではINTEGER（0/1） |
| TIMESTAMP / DATETIME | str | ISO 8601文字列として扱う |
| NULL許容カラム | `T \| None` | Optional表記 |

## 出力形式

生成するモデル構成：

```python
models.py
├── import文
│   ├── from __future__ import annotations
│   ├── from dataclasses import dataclass, field, asdict
│   └── from typing import Any
├── モデルクラス（テーブルごと）
│   ├── @dataclass
│   ├── フィールド定義（型ヒント付き）
│   ├── from_dict(cls, data) -> Self
│   ├── from_row(cls, row) -> Self
│   ├── to_dict(self) -> dict
│   ├── バリデーションプロパティ
│   └── __post_init__(self)（バリデーション）
└── ヘルパー関数
    └── create_table_sql(cls) -> str
```

## 実装パターン

### SQLiteスキーマからの生成手順

```python
import sqlite3
from pathlib import Path


def get_table_schema(db_path: str, table_name: str) -> list[dict]:
    """SQLiteテーブルのスキーマ情報を取得する。"""
    conn = sqlite3.connect(db_path)
    try:
        cursor = conn.execute(f"PRAGMA table_info({table_name})")
        columns = []
        for row in cursor:
            columns.append({
                "cid": row[0],
                "name": row[1],
                "type": row[2],
                "notnull": bool(row[3]),
                "default": row[4],
                "pk": bool(row[5]),
            })
        return columns
    finally:
        conn.close()
```

### dataclass モデルテンプレート

```python
from __future__ import annotations

from dataclasses import dataclass, field, asdict
from typing import Any, Self


@dataclass
class User:
    """usersテーブルのデータモデル。"""

    id: int | None = None
    name: str = ""
    email: str = ""
    age: int = 0
    is_active: bool = True
    created_at: str = ""

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> Self:
        """辞書からインスタンスを生成する。

        未知のキーは無視する。
        """
        valid_keys = {f.name for f in cls.__dataclass_fields__.values()}
        filtered = {k: v for k, v in data.items() if k in valid_keys}
        return cls(**filtered)

    @classmethod
    def from_row(cls, row: tuple, columns: list[str]) -> Self:
        """SQLiteの行タプルからインスタンスを生成する。

        Args:
            row: fetchone()の結果タプル
            columns: cursor.descriptionから取得したカラム名リスト
        """
        return cls.from_dict(dict(zip(columns, row)))

    def to_dict(self) -> dict[str, Any]:
        """辞書に変換する。Noneのフィールドは除外。"""
        return {k: v for k, v in asdict(self).items() if v is not None}

    def to_insert_dict(self) -> dict[str, Any]:
        """INSERT用の辞書（id=NoneならidをAUTOINCREMENTに任せる）。"""
        d = self.to_dict()
        if self.id is None:
            d.pop("id", None)
        return d

    def __post_init__(self) -> None:
        """型変換とバリデーション。"""
        if self.age is not None and isinstance(self.age, str):
            self.age = int(self.age) if self.age else 0
        if isinstance(self.is_active, int):
            self.is_active = bool(self.is_active)
```

### NamedTuple モデルテンプレート

```python
from typing import NamedTuple


class UserRecord(NamedTuple):
    """usersテーブルの読み取り専用レコード。"""

    id: int
    name: str
    email: str
    age: int
    is_active: bool
    created_at: str

    @classmethod
    def from_dict(cls, data: dict) -> "UserRecord":
        return cls(**{k: data[k] for k in cls._fields if k in data})
```

### CREATE TABLE SQL生成

```python
    @classmethod
    def create_table_sql(cls, table_name: str = "users") -> str:
        """CREATE TABLE文を生成する。"""
        type_map = {
            int: "INTEGER",
            float: "REAL",
            str: "TEXT",
            bool: "INTEGER",
            bytes: "BLOB",
        }
        lines = []
        for name, f in cls.__dataclass_fields__.items():
            origin = getattr(f.type, "__origin__", None)
            # int | None のようなUnion型を処理
            if origin is type(int | None):
                args = [a for a in f.type.__args__ if a is not type(None)]
                sql_type = type_map.get(args[0], "TEXT") if args else "TEXT"
                nullable = "NULL"
            else:
                actual = f.type if isinstance(f.type, type) else str
                sql_type = type_map.get(actual, "TEXT")
                nullable = "NOT NULL"

            col_def = f"    {name} {sql_type} {nullable}"
            if name == "id":
                col_def = f"    {name} INTEGER PRIMARY KEY AUTOINCREMENT"
            lines.append(col_def)

        cols = ",\n".join(lines)
        return f"CREATE TABLE IF NOT EXISTS {table_name} (\n{cols}\n);"
```

## サンプル出力

### SQLiteテーブルから生成した例

```python
"""
Data models for rotation-planner database.

Auto-generated by dataclass-model-generator skill.
Generated: 2026-02-07
Source: data/rotation_planner.db
"""

from __future__ import annotations

from dataclasses import dataclass, asdict
from typing import Any, Self


@dataclass
class CropPolygon:
    """crop_polygonsテーブルのデータモデル。"""

    id: int | None = None
    user_id: int = 0
    year: int = 0
    crop_name: str = ""
    geometry: str = ""  # GeoJSON文字列
    area_ha: float = 0.0
    notes: str = ""

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> Self:
        valid_keys = {f.name for f in cls.__dataclass_fields__.values()}
        filtered = {k: v for k, v in data.items() if k in valid_keys}
        return cls(**filtered)

    @classmethod
    def from_row(cls, row: tuple, columns: list[str]) -> Self:
        return cls.from_dict(dict(zip(columns, row)))

    def to_dict(self) -> dict[str, Any]:
        return {k: v for k, v in asdict(self).items() if v is not None}

    def to_insert_dict(self) -> dict[str, Any]:
        d = self.to_dict()
        if self.id is None:
            d.pop("id", None)
        return d

    @property
    def is_valid(self) -> bool:
        """基本バリデーション。"""
        return bool(self.crop_name and self.geometry and self.area_ha > 0)
```

## 注意事項

### 必須要素

- 型ヒント完備（`from __future__ import annotations` 使用）
- from_dict / to_dict 変換メソッド
- docstring付き
- SQLiteのNULL許容に対応した `T | None` 型

### 推奨要素

- `__post_init__` での型変換・バリデーション
- `from_row` メソッド（cursor結果からの変換）
- `to_insert_dict` メソッド（AUTOINCREMENT対応）
- `create_table_sql` クラスメソッド
- `is_valid` プロパティ

### 避けるべき実装

- `Any` 型の多用（できるだけ具体的な型を使う）
- ミュータブルなデフォルト値を直接指定（`field(default_factory=list)` を使用）
- SQLインジェクションの危険がある文字列フォーマット
