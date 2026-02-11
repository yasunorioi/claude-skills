# Frontend-Backend Schema Migration - Skill Definition

**Skill ID**: `frontend-backend-schema-migration`
**Category**: Full-Stack Development / Schema Management
**Version**: 1.0.0
**Created**: 2026-02-11

---

## Overview

React + FastAPI + SQLite 構成でDBスキーマ変更を行った際に、全レイヤーを一貫して修正するパターン。スキーマ不一致によるランタイムエラーを防ぐ。

---

## Use Cases

- DBテーブルのカラム追加/削除/リネーム
- API Pydanticモデルのフィールド変更
- Repositoryクラスの INSERT/UPDATE/SELECT文の修正
- フロントエンドのフォーム/テーブル/APIコールの対応修正
- テストコードの期待値修正

---

## Skill Input

1. **変更内容**: どのテーブルのどのカラムをどう変更するか
2. **プロジェクトルート**: 対象プロジェクトのパス
3. **技術スタック**: DB(SQLite/PostgreSQL) + API(FastAPI/Flask) + Frontend(React/Vue)

---

## Skill Output

修正が必要な全ファイルのリストと、各ファイルの修正内容

---

## Implementation Pattern

### 修正順序（ボトムアップ）

必ず以下の順序で修正する。上流から下流へ。

```
Step 1: db_schema.sql        — DDL修正（CREATE TABLE文）
Step 2: db_access.py         — Repository（INSERT/UPDATE/SELECT文）
Step 3: api/main.py          — Pydanticモデル + エンドポイント
Step 4: frontend/api.js      — APIコール（パラメータ名）
Step 5: frontend/Page.jsx    — UI（フォーム/テーブルカラム）
Step 6: tests/               — テストコードの期待値修正
Step 7: E2E tests            — Page Object + テストデータ修正
```

### Step 1: db_schema.sql

```sql
-- BEFORE: 旧スキーマ
CREATE TABLE IF NOT EXISTS pesticide_masters (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pesticide_name TEXT NOT NULL,    -- 旧カラム名
    crop TEXT NOT NULL,
    target TEXT,
    ...
);

-- AFTER: 新スキーマ
CREATE TABLE IF NOT EXISTS pesticide_masters (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,              -- 新カラム名
    crop TEXT,                       -- NOT NULL制約緩和
    category TEXT,                   -- 新規追加
    manufacturer TEXT,               -- 新規追加
    ...
);
```

### Step 2: Repository (db_access.py)

全てのSQL文を新カラム名に合わせる：

```python
# INSERT文
conn.execute("""
    INSERT INTO pesticide_masters
    (org_id, name, crop, category, manufacturer, ...)
    VALUES (?, ?, ?, ?, ?, ...)
""", (...))

# UPDATE文
conn.execute("""
    UPDATE pesticide_masters SET
        name = ?, crop = ?, category = ?, manufacturer = ?, ...
    WHERE id = ?
""", (...))

# SELECT文 — SELECT * を使っている場合はカラム名変更の影響を受けにくいが、
# row_to_dict で返すためキー名が変わる点に注意
```

### Step 3: API Pydanticモデル (api/main.py)

```python
# BEFORE
class PesticideMasterCreate(BaseModel):
    pesticide_name: str    # 旧フィールド名
    crop: str              # 必須
    target_pest: Optional[str] = None

# AFTER
class PesticideMasterCreate(BaseModel):
    name: str              # 新フィールド名
    crop: Optional[str] = None   # Optional化
    category: Optional[str] = None
    manufacturer: Optional[str] = None
```

### Step 4: Frontend API (api.js)

```javascript
// パラメータ名がPydanticモデルと一致していることを確認
export const pesticideMasterApi = {
  create: async (data) => {
    // data = { name, crop, category, manufacturer, ... }
    const res = await api.post('/api/pesticide-masters', data);
    return res.data;
  },
};
```

### Step 5: Frontend UI (Page.jsx)

```jsx
// テーブルヘッダー
<th>農薬名</th>      {/* name */}
<th>対象作物</th>     {/* crop */}
<th>カテゴリ</th>     {/* category — 新規追加 */}
<th>メーカー</th>     {/* manufacturer — 新規追加 */}

// フォーム入力
<input name="name" />         {/* 旧: pesticide_name */}
<input name="crop" />
<select name="category">...</select>  {/* 新規 */}
<input name="manufacturer" />          {/* 新規 */}
```

### Step 6: Unit Tests

```python
# テストデータのキー名を新スキーマに合わせる
test_data = {
    "name": "テスト農薬",      # 旧: "pesticide_name"
    "crop": "じゃがいも",
    "category": "殺虫剤",       # 新規
    "manufacturer": "テストメーカー",  # 新規
}

# アサーションのキー名も修正
assert result["name"] == "テスト農薬"   # 旧: result["pesticide_name"]
```

### Step 7: E2E Tests

```python
# Page Object のロケータ修正
self.name_input = page.locator('input[name="name"]')

# テストデータの修正
pesticide_page.add_pesticide(
    name="E2Eテスト農薬",
    target_crop="じゃがいも",    # ドロップダウン選択肢と一致させる
    category="殺虫剤",
)
```

---

## Checklist

修正完了後に必ず確認：

- [ ] `db_schema.sql` の CREATE TABLE が新スキーマ
- [ ] Repository の INSERT/UPDATE/SELECT が新カラム名
- [ ] Pydantic モデルのフィールド名が Repository と一致
- [ ] フロントエンドの API コールのキー名が Pydantic と一致
- [ ] フロントエンドの UI が新フィールドを表示/入力
- [ ] Unit テストが新キー名でアサート
- [ ] E2E テストの Page Object ロケータが実DOMと一致
- [ ] `pytest` 全件 PASSED

---

## Common Pitfalls

### 1. SELECT * のキー名変更
**問題**: `SELECT *` + `row_to_dict()` で返すため、カラム名変更がAPIレスポンスのキー名に直結する
**解決**: フロントエンドで使用するキー名を全てgrepで洗い出す

### 2. Optional化の罠
**問題**: `NOT NULL` → `Optional` にしたとき、既存データに NULL が入り UI が `null` 表示
**解決**: COALESCE やフロントエンドで `|| ''` でフォールバック

### 3. E2Eテストのロケータ不一致
**問題**: テストのCSSセレクタが実DOMのclassName/構造と合わない
**解決**: React DevTools or `page.content()` で実DOMを確認してからロケータを書く

---

**Skill Author**: 足軽4号提案 / 将軍承認
**Last Updated**: 2026-02-11
