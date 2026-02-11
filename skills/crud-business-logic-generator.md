# CRUD Business Logic Generator

Gradio + SQLiteアプリの標準的なCRUDビジネスロジック層を自動生成するスキル。

- Skill ID: crud-business-logic-generator
- Category: code-generation
- Version: 1.0.0
- Created: 2026-02-07
- Platform: Python 3.10+, Gradio 4.x+

## 概要

Gradioアプリケーションにおける標準的なCRUD（Create/Read/Update/Delete）操作のビジネスロジック層を自動生成する。SQLiteデータベースに対するトランザクション管理、エラーハンドリング、バリデーション、Gradioイベントハンドラとの連携パターンを含む実用的なコードを出力する。

## 使用方法

ユーザーが以下のいずれかを要求したら、このスキルを使用：

- Gradio + SQLiteアプリのCRUD機能を実装したい
- ビジネスロジック層のテンプレートが欲しい
- SQLiteのCRUD関数をGradioに接続したい
- Gradioアプリのデータ管理層を設計したい

## 入力パラメータ

| 項目 | 説明 | 例 | 必須 |
|------|------|-----|------|
| entity_name | エンティティ名 | "crop_polygon" | Yes |
| table_name | SQLiteテーブル名 | "crop_polygons" | Yes |
| fields | フィールド定義 | [{"name": "crop_name", "type": "str", "required": true}] | Yes |
| db_path | データベースパス | "data/app.db" | No (default: configで指定) |
| include_gradio | Gradioハンドラを含む | true | No (default: true) |
| include_search | 検索機能を含む | true | No (default: false) |
| user_scoped | ユーザースコープ | true | No (default: false) |

## 出力形式

生成するモジュール構成：

```
{entity}_service.py
├── import文
├── 例外クラス
│   ├── ServiceError (基底)
│   ├── NotFoundError
│   └── ValidationError
├── バリデーション関数
│   └── validate_{entity}(data) -> list[str]
├── CRUD関数
│   ├── create_{entity}(db_path, data) -> int
│   ├── get_{entity}(db_path, id) -> dict | None
│   ├── list_{entities}(db_path, ...) -> list[dict]
│   ├── update_{entity}(db_path, id, data) -> bool
│   └── delete_{entity}(db_path, id) -> bool
├── Gradioイベントハンドラ
│   ├── on_create(...) -> tuple[str, DataFrame]
│   ├── on_update(...) -> tuple[str, DataFrame]
│   ├── on_delete(...) -> tuple[str, DataFrame]
│   └── on_refresh(...) -> DataFrame
└── ヘルパー関数
    └── _get_connection(db_path) -> sqlite3.Connection
```

## 実装パターン

### コネクション管理

```python
from __future__ import annotations

import sqlite3
from contextlib import contextmanager
from pathlib import Path
from typing import Any, Generator


@contextmanager
def _get_connection(
    db_path: str | Path,
    readonly: bool = False,
) -> Generator[sqlite3.Connection, None, None]:
    """SQLiteコネクションのコンテキストマネージャ。

    - WALモード有効（読み書き並行性向上）
    - foreign_keys有効
    - row_factory = sqlite3.Row（カラム名アクセス）
    """
    uri = f"file:{db_path}?mode=ro" if readonly else str(db_path)
    conn = sqlite3.connect(uri, uri=readonly)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    try:
        yield conn
        if not readonly:
            conn.commit()
    except Exception:
        if not readonly:
            conn.rollback()
        raise
    finally:
        conn.close()
```

### 例外クラス

```python
class ServiceError(Exception):
    """ビジネスロジック層の基底例外。"""


class NotFoundError(ServiceError):
    """対象レコードが見つからない。"""

    def __init__(self, entity: str, id: int):
        super().__init__(f"{entity} (id={id}) not found")
        self.entity = entity
        self.id = id


class ValidationError(ServiceError):
    """バリデーションエラー。"""

    def __init__(self, errors: list[str]):
        super().__init__(f"Validation failed: {'; '.join(errors)}")
        self.errors = errors
```

### バリデーション

```python
def validate_crop_polygon(data: dict[str, Any]) -> list[str]:
    """入力データのバリデーション。

    Returns:
        エラーメッセージのリスト（空リスト=OK）
    """
    errors: list[str] = []

    if not data.get("crop_name", "").strip():
        errors.append("作物名は必須です")

    if not data.get("geometry", "").strip():
        errors.append("ポリゴンデータは必須です")

    area = data.get("area_ha")
    if area is not None:
        try:
            area_val = float(area)
            if area_val <= 0:
                errors.append("面積は正の値を指定してください")
        except (ValueError, TypeError):
            errors.append("面積は数値で指定してください")

    return errors
```

### CRUD関数

```python
def create_crop_polygon(
    db_path: str | Path,
    data: dict[str, Any],
) -> int:
    """レコードを作成し、生成されたIDを返す。

    Args:
        db_path: データベースパス
        data: 作成するデータ

    Returns:
        作成されたレコードのID

    Raises:
        ValidationError: バリデーション失敗
    """
    errors = validate_crop_polygon(data)
    if errors:
        raise ValidationError(errors)

    with _get_connection(db_path) as conn:
        cursor = conn.execute(
            """
            INSERT INTO crop_polygons (crop_name, geometry, area_ha, year, notes)
            VALUES (:crop_name, :geometry, :area_ha, :year, :notes)
            """,
            {
                "crop_name": data["crop_name"].strip(),
                "geometry": data["geometry"],
                "area_ha": float(data.get("area_ha", 0)),
                "year": int(data.get("year", 0)),
                "notes": data.get("notes", ""),
            },
        )
        return cursor.lastrowid


def get_crop_polygon(
    db_path: str | Path,
    polygon_id: int,
) -> dict[str, Any] | None:
    """IDでレコードを取得する。"""
    with _get_connection(db_path, readonly=True) as conn:
        row = conn.execute(
            "SELECT * FROM crop_polygons WHERE id = ?",
            (polygon_id,),
        ).fetchone()
        return dict(row) if row else None


def list_crop_polygons(
    db_path: str | Path,
    year: int | None = None,
    limit: int = 100,
    offset: int = 0,
) -> list[dict[str, Any]]:
    """レコード一覧を取得する。"""
    query = "SELECT * FROM crop_polygons"
    params: list[Any] = []

    if year is not None:
        query += " WHERE year = ?"
        params.append(year)

    query += " ORDER BY id DESC LIMIT ? OFFSET ?"
    params.extend([limit, offset])

    with _get_connection(db_path, readonly=True) as conn:
        rows = conn.execute(query, params).fetchall()
        return [dict(row) for row in rows]


def update_crop_polygon(
    db_path: str | Path,
    polygon_id: int,
    data: dict[str, Any],
) -> bool:
    """レコードを更新する。

    Returns:
        更新が成功したかどうか

    Raises:
        NotFoundError: 対象が見つからない
        ValidationError: バリデーション失敗
    """
    errors = validate_crop_polygon(data)
    if errors:
        raise ValidationError(errors)

    with _get_connection(db_path) as conn:
        cursor = conn.execute(
            """
            UPDATE crop_polygons
            SET crop_name = :crop_name,
                geometry = :geometry,
                area_ha = :area_ha,
                year = :year,
                notes = :notes
            WHERE id = :id
            """,
            {
                "id": polygon_id,
                "crop_name": data["crop_name"].strip(),
                "geometry": data["geometry"],
                "area_ha": float(data.get("area_ha", 0)),
                "year": int(data.get("year", 0)),
                "notes": data.get("notes", ""),
            },
        )
        if cursor.rowcount == 0:
            raise NotFoundError("crop_polygon", polygon_id)
        return True


def delete_crop_polygon(
    db_path: str | Path,
    polygon_id: int,
) -> bool:
    """レコードを削除する。

    Raises:
        NotFoundError: 対象が見つからない
    """
    with _get_connection(db_path) as conn:
        cursor = conn.execute(
            "DELETE FROM crop_polygons WHERE id = ?",
            (polygon_id,),
        )
        if cursor.rowcount == 0:
            raise NotFoundError("crop_polygon", polygon_id)
        return True
```

### Gradioイベントハンドラ

```python
import pandas as pd


def _make_status(message: str, is_error: bool = False) -> str:
    """ステータスメッセージを生成する。"""
    prefix = "エラー: " if is_error else ""
    return f"{prefix}{message}"


def _refresh_table(db_path: str | Path, **filters) -> pd.DataFrame:
    """テーブルデータを再取得してDataFrameにする。"""
    rows = list_crop_polygons(db_path, **filters)
    if not rows:
        return pd.DataFrame()
    return pd.DataFrame(rows)


def on_create(
    db_path: str | Path,
    crop_name: str,
    geometry: str,
    area_ha: float,
    year: int,
    notes: str = "",
) -> tuple[str, pd.DataFrame]:
    """Gradio Create ボタンのハンドラ。

    Returns:
        (ステータスメッセージ, 更新後テーブルデータ)
    """
    try:
        new_id = create_crop_polygon(db_path, {
            "crop_name": crop_name,
            "geometry": geometry,
            "area_ha": area_ha,
            "year": year,
            "notes": notes,
        })
        return (
            _make_status(f"作成しました (ID: {new_id})"),
            _refresh_table(db_path),
        )
    except ValidationError as e:
        return _make_status(str(e), is_error=True), _refresh_table(db_path)
    except Exception as e:
        return _make_status(f"予期せぬエラー: {e}", is_error=True), _refresh_table(db_path)


def on_delete(
    db_path: str | Path,
    polygon_id: int,
) -> tuple[str, pd.DataFrame]:
    """Gradio Delete ボタンのハンドラ。"""
    try:
        delete_crop_polygon(db_path, polygon_id)
        return (
            _make_status(f"削除しました (ID: {polygon_id})"),
            _refresh_table(db_path),
        )
    except NotFoundError:
        return _make_status(f"ID {polygon_id} が見つかりません", is_error=True), _refresh_table(db_path)
    except Exception as e:
        return _make_status(f"予期せぬエラー: {e}", is_error=True), _refresh_table(db_path)
```

### Gradio UI接続例

```python
import gradio as gr

with gr.Blocks() as app:
    status = gr.Textbox(label="ステータス", interactive=False)
    table = gr.DataFrame(label="一覧")

    with gr.Row():
        crop_name = gr.Textbox(label="作物名")
        area_ha = gr.Number(label="面積 (ha)")
        year = gr.Number(label="年度", precision=0)

    create_btn = gr.Button("作成")
    create_btn.click(
        fn=lambda *args: on_create(DB_PATH, *args),
        inputs=[crop_name, gr.Textbox(visible=False), area_ha, year],
        outputs=[status, table],
    )

    app.load(
        fn=lambda: _refresh_table(DB_PATH),
        outputs=[table],
    )
```

## サンプル出力

### 最小構成のサービスモジュール

```python
"""
CRUD business logic for crop_polygons table.

Auto-generated by crud-business-logic-generator skill.
Generated: 2026-02-07
"""

from __future__ import annotations

import sqlite3
from contextlib import contextmanager
from pathlib import Path
from typing import Any, Generator


class ServiceError(Exception):
    pass

class NotFoundError(ServiceError):
    def __init__(self, entity: str, id: int):
        super().__init__(f"{entity} (id={id}) not found")

class ValidationError(ServiceError):
    def __init__(self, errors: list[str]):
        super().__init__("; ".join(errors))
        self.errors = errors


@contextmanager
def _get_connection(db_path: str | Path) -> Generator[sqlite3.Connection, None, None]:
    conn = sqlite3.connect(str(db_path))
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()


# --- Validation ---

def validate(data: dict[str, Any]) -> list[str]:
    errors = []
    if not data.get("crop_name", "").strip():
        errors.append("作物名は必須です")
    return errors


# --- CRUD ---

def create(db_path, data):
    errs = validate(data)
    if errs:
        raise ValidationError(errs)
    with _get_connection(db_path) as conn:
        cur = conn.execute(
            "INSERT INTO crop_polygons (crop_name, area_ha) VALUES (?, ?)",
            (data["crop_name"], data.get("area_ha", 0)),
        )
        return cur.lastrowid

def get(db_path, id):
    with _get_connection(db_path) as conn:
        row = conn.execute("SELECT * FROM crop_polygons WHERE id=?", (id,)).fetchone()
        return dict(row) if row else None

def list_all(db_path, limit=100):
    with _get_connection(db_path) as conn:
        return [dict(r) for r in conn.execute(
            "SELECT * FROM crop_polygons ORDER BY id DESC LIMIT ?", (limit,)
        )]

def update(db_path, id, data):
    errs = validate(data)
    if errs:
        raise ValidationError(errs)
    with _get_connection(db_path) as conn:
        cur = conn.execute(
            "UPDATE crop_polygons SET crop_name=?, area_ha=? WHERE id=?",
            (data["crop_name"], data.get("area_ha", 0), id),
        )
        if cur.rowcount == 0:
            raise NotFoundError("crop_polygon", id)

def delete(db_path, id):
    with _get_connection(db_path) as conn:
        cur = conn.execute("DELETE FROM crop_polygons WHERE id=?", (id,))
        if cur.rowcount == 0:
            raise NotFoundError("crop_polygon", id)
```

## 注意事項

### 必須要素

- コンテキストマネージャによるコネクション管理（with文）
- トランザクション管理（commit/rollback）
- バリデーション関数の分離
- パラメータ化クエリ（SQLインジェクション防止）
- 型ヒント完備

### 推奨要素

- WALモード有効化（読み書き並行性向上）
- foreign_keys PRAGMA有効化
- row_factory = sqlite3.Row（カラム名アクセス）
- Gradioイベントハンドラの戻り値は `tuple[str, DataFrame]`
- NotFoundError/ValidationErrorの分離

### 避けるべき実装

- 文字列フォーマットによるSQL組み立て（SQLインジェクション）
- コネクションの使い回し（Gradioはマルチスレッド）
- try-except-pass（エラーの握りつぶし）
- グローバル変数でのDB接続保持
