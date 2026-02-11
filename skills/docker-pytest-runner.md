# Docker Pytest Runner - Skill Definition

**Skill ID**: `docker-pytest-runner`
**Category**: Testing / CI
**Version**: 1.0.0
**Created**: 2026-02-11

---

## Overview

Docker内でpytestを実行するパターン。ローカル環境にMeCab、libmecab-dev等のシステムライブラリがインストールされていないプロジェクトで、Dockerコンテナ内でテストを実行する。

---

## Use Cases

- MeCab/形態素解析ライブラリに依存するテスト
- 特定のOSパッケージ（libxml2, libpq-dev等）に依存するテスト
- Python以外のシステム依存（ImageMagick, ffmpeg等）があるテスト
- CI環境と同一の環境でローカルテストしたい場合

---

## Skill Input

1. **プロジェクトパス**: テスト対象のプロジェクトルートディレクトリ
2. **テスト対象**: 特定テストファイル/ディレクトリ or 全体
3. **システム依存**: 必要なaptパッケージ一覧

---

## Skill Output

1. テスト用Dockerfile（既存があれば流用）
2. docker-compose.ymlのtestサービス定義
3. テスト実行コマンド

---

## Implementation Pattern

### Step 1: Dockerfileの確認/作成

プロジェクトに既存のDockerfileがあるか確認。なければ作成：

```dockerfile
# Dockerfile.test
FROM python:3.12-slim

# システム依存パッケージ
RUN apt-get update && apt-get install -y --no-install-recommends \
    mecab \
    libmecab-dev \
    mecab-ipadic-utf8 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 依存インストール
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install --no-cache-dir pytest pytest-cov

# ソースコピー
COPY . .

# テスト実行
CMD ["python", "-m", "pytest", "-v", "--tb=short"]
```

### Step 2: docker-compose.yml にtestサービス追加

```yaml
services:
  test:
    build:
      context: .
      dockerfile: Dockerfile.test
    volumes:
      - .:/app  # ソースのライブマウント（開発時）
    environment:
      - PYTHONPATH=/app
    command: python -m pytest tests/ -v --tb=short
```

### Step 3: 実行コマンド

```bash
# ビルド＆全テスト実行
docker compose run --rm test

# 特定テストのみ
docker compose run --rm test python -m pytest tests/test_specific.py -v

# カバレッジ付き
docker compose run --rm test python -m pytest tests/ --cov=src --cov-report=term-missing

# キャッシュ活用（2回目以降高速）
docker compose build test && docker compose run --rm test
```

---

## Best Practices

### ボリュームマウント vs COPY

| 方式 | 用途 | メリット |
|------|------|---------|
| `volumes: .:/app` | 開発時 | ソース変更が即反映、リビルド不要 |
| `COPY . .` | CI/本番 | 再現性が高い、キャッシュ効率良い |

### レイヤーキャッシュの活用

```dockerfile
# requirements.txtを先にコピーしてキャッシュ効率を上げる
COPY requirements.txt .
RUN pip install -r requirements.txt
# ソースは後でコピー（変更頻度が高いため）
COPY . .
```

### テスト結果の永続化

```yaml
services:
  test:
    volumes:
      - ./test-results:/app/test-results
    command: >
      python -m pytest tests/ -v
      --junitxml=test-results/junit.xml
      --cov-report=html:test-results/coverage
```

---

## Common Pitfalls

### 1. MeCab辞書パスの問題
**問題**: `mecab-ipadic-utf8` をインストールしてもMeCabが辞書を見つけられない
**解決**: `mecabrc` の辞書パスを確認、必要なら環境変数 `MECABRC` を設定

### 2. テストDBのパーミッション
**問題**: コンテナ内でSQLiteファイルが書き込めない
**解決**: `/tmp` や `tmpdir` fixture を使用、またはボリュームの所有者を合わせる

### 3. ネットワーク依存テストの分離
**問題**: 外部APIに依存するテストがDockerコンテナ内で失敗する
**解決**: `pytest.mark.integration` でマーク、デフォルトは `-m "not integration"` で実行

---

**Skill Author**: 部屋子3号提案 / 将軍承認
**Last Updated**: 2026-02-11
