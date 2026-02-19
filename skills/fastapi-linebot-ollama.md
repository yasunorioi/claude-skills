# FastAPI LINE Bot + Ollama - Skill Definition

**Skill ID**: `fastapi-linebot-ollama`
**Category**: Chatbot / LLM Integration
**Version**: 1.0.0
**Created**: 2026-02-20

---

## Overview

FastAPI + line-bot-sdk v3 + Ollama を使ったLINE Bot壁打ちパターン。
ローカルLLM（Ollama）とLINEを繋ぐBot雛形で、**クラウドAPIコストゼロ**で任意のLLMをLINE Bot化できる。

VPS上にDocker Composeで `ollama` + `linebot` の2サービスを立てるだけで完結する。
SQLite会話ログ・システムプロンプトのカスタマイズ・ヘルスチェックを標準装備。

---

## Use Cases

- 特定ドメインの壁打ち相手をLINE Bot化したい（農業アドバイス、技術相談、語学練習等）
- OpenAI等のクラウドAPIコストを避け、自前LLMをLINEに繋ぎたい
- VPS + Dockerがある環境でLINE Botをゼロから立てたい
- 会話ログをSQLiteで手元に蓄積したい

### 適用実績

- unipi-agri-ha プロジェクト 農業ハウス管理LINE Bot（subtask_486 / subtask_511）:
  - モデル: qwen3:8b（VPS CPU環境で動作確認済み）
  - システムプロンプト: 恵庭ハウスのセンサー分析知見を組み込み
  - SQLite会話ログ + Dockerボリューム永続化
  - ngrokでLINE Webhook受信（開発・本番兼用）

---

## Skill Input

1. **LINE Bot認証情報**: `LINE_CHANNEL_SECRET`, `LINE_CHANNEL_ACCESS_TOKEN`（LINE Developer Consoleで取得）
2. **Ollamaモデル名**: 使用するモデル（例: `qwen2.5:1.5b`, `qwen3:8b`, `llama3.2`）
3. **システムプロンプト**: Botの役割・知識・応答ルールを定義したテキスト
4. **デプロイ先**: VPS（SSH可能なLinuxサーバー）または localhost（開発時）

---

## Skill Output

以下のファイル一式（最小構成 5ファイル）:

```
linebot/
├── app.py                    # FastAPI Webhookハンドラ
├── llm_client.py             # Ollama APIクライアント
├── system_prompt.py          # システムプロンプト定義
├── requirements.txt          # Python依存関係
├── Dockerfile                # コンテナビルド設定
├── docker-compose.override.yml  # ollama + linebotサービス定義
└── .env.example              # 環境変数テンプレート
```

---

## Implementation Pattern

### Step 1: ディレクトリ構造の作成

```bash
mkdir -p linebot
cd linebot
```

### Step 2: 依存関係 (`requirements.txt`)

```
line-bot-sdk>=3.0
fastapi>=0.100.0
uvicorn>=0.23.0
httpx>=0.24.0
```

### Step 3: システムプロンプト (`system_prompt.py`)

ここがBotの「個性」の核心部分。ドメイン固有の知識・応答ルールを記述する。

```python
"""システムプロンプト定義"""

SYSTEM_PROMPT = """あなたは[ドメイン名]の専門アシスタントです。
[ドメイン固有の知識・ルールをここに記述]

## 回答のルール
- 平易な言葉で答える
- 短く具体的に（3〜5文程度）
- 分からないことは素直に「分かりません」と答える
"""

def get_system_prompt() -> str:
    return SYSTEM_PROMPT
```

### Step 4: Ollama APIクライアント (`llm_client.py`)

```python
"""
Ollama API クライアント

http://ollama:11434/api/chat に httpx でリクエストを送る。
WebhookHandlerが同期ハンドラのみ対応のため、同期版（generate_response_sync）を使用。
"""

import os
import httpx
from system_prompt import get_system_prompt

OLLAMA_URL = os.getenv("OLLAMA_URL", "http://ollama:11434")
MODEL_NAME = os.getenv("MODEL_NAME", "qwen2.5:1.5b")
TIMEOUT_SEC = 60.0
MAX_RETRIES = 1


def generate_response_sync(user_message: str) -> str:
    """
    ユーザーメッセージをOllamaに送り、応答テキストを返す（同期版）。

    LINE WebhookHandlerは同期ハンドラのみサポートするため、
    asyncではなく同期httpxクライアントを使用する。
    """
    payload = {
        "model": MODEL_NAME,
        "messages": [
            {"role": "system", "content": get_system_prompt()},
            {"role": "user", "content": user_message},
        ],
        "stream": False,      # LINE replyは一括応答なのでstream不要
        "think": False,       # thinking modeはLINEには不要（長くなるだけ）
        "options": {
            "temperature": 0.7,
            "top_p": 0.9,
            "num_predict": 512,
        },
    }

    for attempt in range(MAX_RETRIES + 1):
        try:
            with httpx.Client(timeout=TIMEOUT_SEC) as client:
                resp = client.post(f"{OLLAMA_URL}/api/chat", json=payload)
                resp.raise_for_status()
                return resp.json().get("message", {}).get("content", "").strip()
        except httpx.TimeoutException:
            if attempt < MAX_RETRIES:
                continue
            raise
        except httpx.HTTPStatusError as e:
            raise RuntimeError(f"Ollama API error: {e.response.status_code}") from e

    return ""


async def check_ollama_health() -> bool:
    """起動時にOllamaの疎通を確認"""
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            resp = await client.get(f"{OLLAMA_URL}/api/tags")
            return resp.status_code == 200
    except Exception:
        return False
```

### Step 5: FastAPI Webhookハンドラ (`app.py`)

```python
"""
LINE Bot Webhook Handler — FastAPI

LINE Webhook → Ollama (任意モデル) → LINE Reply
SQLite会話ログ付き。
"""

import os
import sqlite3
import logging
from contextlib import asynccontextmanager
from datetime import datetime, timezone

from fastapi import FastAPI, HTTPException, Request
from linebot.v3 import WebhookHandler
from linebot.v3.exceptions import InvalidSignatureError
from linebot.v3.messaging import (
    ApiClient, MessagingApi, Configuration,
    ReplyMessageRequest, TextMessage,
)
from linebot.v3.webhooks import MessageEvent, TextMessageContent

from llm_client import generate_response_sync, check_ollama_health, MODEL_NAME

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

LINE_CHANNEL_SECRET = os.environ["LINE_CHANNEL_SECRET"]
LINE_CHANNEL_ACCESS_TOKEN = os.environ["LINE_CHANNEL_ACCESS_TOKEN"]

configuration = Configuration(access_token=LINE_CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(LINE_CHANNEL_SECRET)

DB_PATH = os.getenv("CONVERSATION_DB_PATH", "/app/data/conversations.db")


def init_db() -> None:
    """SQLite DBを初期化（アプリ起動時に1回だけ呼ぶ）"""
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS conversations (
                id         INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp  TEXT NOT NULL,
                user_id    TEXT NOT NULL,
                role       TEXT NOT NULL,
                message    TEXT NOT NULL,
                model      TEXT NOT NULL,
                session_id TEXT
            )
        """)
        conn.commit()


def log_message(user_id: str, role: str, message: str, model: str) -> None:
    """会話ログをSQLiteに記録。失敗してもメッセージ処理を妨げない"""
    try:
        with sqlite3.connect(DB_PATH) as conn:
            conn.execute(
                "INSERT INTO conversations (timestamp, user_id, role, message, model) VALUES (?, ?, ?, ?, ?)",
                (datetime.now(timezone.utc).isoformat(), user_id, role, message, model),
            )
            conn.commit()
    except Exception as e:
        logger.warning(f"Failed to log message: {e}")


@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()
    ok = await check_ollama_health()
    logger.info(f"Ollama health: {'up' if ok else 'DOWN - responses will fail'}")
    yield


app = FastAPI(title="linebot", lifespan=lifespan)


@app.get("/health")
async def health():
    ollama_ok = await check_ollama_health()
    return {"status": "ok", "ollama": "up" if ollama_ok else "down"}


@app.post("/callback")
async def callback(request: Request):
    """LINE Webhookを受信してhandlerに渡す"""
    signature = request.headers.get("X-Line-Signature", "")
    body = await request.body()
    try:
        handler.handle(body.decode("utf-8"), signature)
    except InvalidSignatureError:
        raise HTTPException(status_code=400, detail="Invalid signature")
    return {"status": "ok"}


@handler.add(MessageEvent, message=TextMessageContent)
def handle_text_message(event: MessageEvent):
    """
    テキストメッセージ受信 → Ollama → LINE Reply。

    line-bot-sdk v3 の WebhookHandler は同期ハンドラのみ対応。
    async def にしてはいけない（イベントが無視される）。
    """
    user_text = event.message.text
    user_id = event.source.user_id or "unknown"

    log_message(user_id=user_id, role="user", message=user_text, model=MODEL_NAME)

    try:
        reply = generate_response_sync(user_text)
        if not reply:
            reply = "申し訳ありません、応答を生成できませんでした。"
    except Exception as e:
        logger.error(f"LLM error: {e}")
        reply = "エラーが発生しました。しばらくしてから再度お試しください。"

    with ApiClient(configuration) as api_client:
        MessagingApi(api_client).reply_message(
            ReplyMessageRequest(
                reply_token=event.reply_token,
                messages=[TextMessage(text=reply)],
            )
        )

    log_message(user_id=user_id, role="assistant", message=reply, model=MODEL_NAME)
```

### Step 6: Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# .env は実行時に env_file で渡す（コンテナに含めない）
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8443"]
```

### Step 7: docker-compose 定義 (`docker-compose.override.yml`)

既存の docker-compose.yml に追記する形で使う。

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 3

  linebot:
    build: ./linebot
    ports:
      - "8443:8443"
    env_file:
      - ./linebot/.env
    environment:
      - OLLAMA_URL=http://ollama:11434
    volumes:
      - linebot_data:/app/data      # SQLite会話ログの永続化
    depends_on:
      ollama:
        condition: service_healthy
    restart: unless-stopped

volumes:
  ollama_data:
  linebot_data:
```

### Step 8: 環境変数テンプレート (`.env.example`)

```bash
# LINE Developer Console から取得
# https://developers.line.biz/console/
LINE_CHANNEL_SECRET=your_channel_secret_here
LINE_CHANNEL_ACCESS_TOKEN=your_channel_access_token_here

# Ollama設定（docker-compose内では ollama がサービス名）
OLLAMA_URL=http://ollama:11434
MODEL_NAME=qwen2.5:1.5b

# 会話ログDB（デフォルトのまま使用可）
CONVERSATION_DB_PATH=/app/data/conversations.db
```

---

## デプロイ手順

### VPSへのデプロイ

```bash
# 1. VPS にSSHログイン
ssh user@your-vps

# 2. リポジトリをpull
git pull origin main

# 3. .env を作成して認証情報を記入
cp linebot/.env.example linebot/.env
nano linebot/.env   # LINE_CHANNEL_SECRET と LINE_CHANNEL_ACCESS_TOKEN を記入

# 4. サービス起動
docker compose up -d ollama linebot

# 5. Ollamaモデルをpull（初回のみ、数分かかる）
docker exec $(docker compose ps -q ollama) ollama pull qwen2.5:1.5b

# 6. ヘルスチェック
curl http://localhost:8443/health
# → {"status":"ok","ollama":"up"}
```

### HTTPS対応（LINE Webhook必須）

LINE WebhookはHTTPS必須。以下のいずれかで対応:

```bash
# 開発・テスト: ngrok（最速）
docker run -d --network host \
  -e NGROK_AUTHTOKEN=<your-token> \
  ngrok/ngrok http 8443

# 本番（ドメインあり）: Caddy（自動Let's Encrypt）
# Caddyfile:
# your-domain.com {
#   reverse_proxy localhost:8443
# }
```

### LINE Developer ConsoleでWebhook URL設定

```
https://<your-domain-or-ngrok>/callback
```

---

## テスト方法

### ローカル開発テスト（ngrok）

```bash
# 1. サービス起動（Docker不使用の場合）
pip install -r requirements.txt
LINE_CHANNEL_SECRET=xxx LINE_CHANNEL_ACCESS_TOKEN=xxx \
  OLLAMA_URL=http://localhost:11434 \
  uvicorn app:app --port 8443

# 2. ngrokでトンネル開通
ngrok http 8443

# 3. LINE Developer ConsoleでWebhook URLを更新
# 4. LINEアプリからBotにメッセージ送信 → 返答確認
```

### ヘルスチェック

```bash
# Botとollamaの疎通確認
curl https://<your-domain>/health
# {"status":"ok","ollama":"up"}

# 署名バリデーション確認（InvalidSignatureError が返れば正常）
curl -X POST https://<your-domain>/callback \
  -H "Content-Type: application/json" \
  -d '{"events":[]}'
# {"detail":"Invalid signature"}
```

### 会話ログ確認

```bash
# コンテナ内でSQLite確認
docker exec $(docker compose ps -q linebot) \
  sqlite3 /app/data/conversations.db \
  "SELECT timestamp, user_id, role, substr(message,1,50) FROM conversations ORDER BY id DESC LIMIT 10;"
```

---

## カスタマイズポイント

### モデルの選択指針

| モデル | サイズ | 用途 | VPS必要RAM |
|--------|--------|------|-----------|
| `qwen2.5:1.5b` | 1GB | 軽量・高速・日本語対応 | 2GB以上 |
| `qwen3:8b` | 5GB | 高品質・日本語対応 | 8GB以上 |
| `llama3.2:3b` | 2GB | 英語特化・軽量 | 4GB以上 |
| `gemma3:4b` | 3GB | バランス型 | 6GB以上 |

### システムプロンプトのカスタマイズ

`system_prompt.py` の `SYSTEM_PROMPT` を書き換えるだけでBotの個性を変えられる。
以下のドメインで実績あり:
- 農業ハウス管理アドバイザー
- 語学練習（英会話、中国語等）
- 技術ドキュメント Q&A

### キーワードショートカットの追加

`app.py` の `handle_text_message` 内に追加:

```python
# 例: 「今日の天気」→ 固定応答
if user_text.strip() == "今日の天気":
    reply = "天気情報は現在取得できません。"
else:
    reply = generate_response_sync(user_text)
```

---

## Notes / Limitations

- **LINE Webhookのタイムアウト**: LINE側のタイムアウトは約5秒。Ollamaの応答が遅い場合（大きいモデル・低スペックVPS）はタイムアウトが発生する。軽量モデルを選ぶか、VPSのスペックを上げること
- **WebhookHandlerは同期のみ**: `line-bot-sdk v3` の `WebhookHandler` は同期ハンドラのみ対応。`async def handle_text_message` にすると**イベントが無視されてBotが応答しない**。必ず `def`（同期）にすること
- **think: False**: qwen3等のthinking modeを持つモデルは `"think": False` を指定しないとLINEの文字数制限（5000文字）を超えることがある
- **HTTPS必須**: LINE Webhookの仕様上、Webhook URLはHTTPS必須。開発時はngrok、本番はCaddy/Let's Encryptを使うこと
- **メモリ**: Ollamaはモデルをメモリにロードし続ける。VPSのメモリ残量をモデルサイズに合わせて確認すること（qwen2.5:1.5b: 約1.5GB）
- **SQLiteの競合**: 高トラフィック環境では複数スレッドからのSQLite書き込み競合が発生する可能性あり。1日数十〜数百メッセージ程度なら問題なし
- **会話コンテキスト**: 本実装はステートレス（1ターンのみ）。複数ターンの文脈を保持するには、DB読み込み+メッセージ配列構築が必要（未実装）

---

**Skill Author**: 足軽1号（subtask_511/subtask_486の成果を設計書化）
**Last Updated**: 2026-02-20
