# MCP Server Scaffold (Python) - Skill Definition

**Skill ID**: `mcp-server-scaffold-python`
**Category**: Developer Tools / AI Integration
**Version**: 1.0.0
**Created**: 2026-02-20

---

## Overview

Python製MCPサーバーの雛形を生成するパターン。MCP仕様(Protocol Revision: 2025-06-18)準拠で、`mcp>=1.0.0`パッケージの`FastMCP`を使用。stdio/Streamable HTTP両トランスポート対応、`@mcp.tool()` デコレータによるツール定義、`pyproject.toml`（hatchling build）付きのプロジェクト骨格を提供する。

Claude Desktop / Claude Code から `uvx <package>` 一発で起動できる構成が最終ゴール。

**主要な設計判断**:
- `FastMCP` を使用（低レベル `Server` クラスより簡潔で、プロダクション実績あり）
- ツール定義は Python型アノテーション → MCP JSON Schema に自動変換
- stdio がデフォルト（Claude Desktop/Code向け）。Streamable HTTPはオプション起動
- `lifespan` コンテキストマネージャで起動/停止処理を管理

---

## Use Cases

- Claude Desktop / Claude Code から外部システム（IoT, DB, API等）を制御するMCPサーバーを新規作成したい
- 既存のPythonスクリプト群をMCPツールとして公開したい
- stdioとStreamable HTTP両方に対応したMCPサーバーを最小工数で構築したい
- MCPサーバーをPyPIパッケージとして公開したい（`uvx`でインストール不要利用）

### 適用実績

- uecs-ccm-mcp（UECS-CCM農業施設制御 × MCP）（subtask_504）:
  - 5ツール（get_sensor_data, get_actuator_status, set_actuator, get_weather_summary, list_nodes）
  - stdioトランスポート + オプションbridgeモード（HTTP API経由）
  - lifespan で UDP multicastレシーバーを起動/停止
  - pytest 17テストケース（実機キャプチャデータ使用）
  - `uvx uecs-ccm-mcp` で Claude Code から直接接続成功

---

## Skill Input

1. **スキル名・パッケージ名**: スネークケース（例: `my-mcp-server` → `my_mcp_server`）
2. **ツール一覧**: 各ツールの名前・説明・入力パラメータ（型付き）・戻り値型
3. **起動時リソース**: lifespan で管理するリソース（DB接続、バックグラウンドタスク等）の有無
4. **トランスポート**: stdio のみ or stdio + Streamable HTTP両対応
5. **PyPI公開要否**: `uvx` 起動に必要。公開する場合はPyPIアカウントが必要

---

## Skill Output

1. **プロジェクト構成**（以下のディレクトリ/ファイル群）:
   - `src/<package_name>/server.py` — MCPサーバーエントリーポイント
   - `src/<package_name>/__init__.py`
   - `tests/test_tools.py` — pytestテストテンプレート
   - `pyproject.toml` — hatchling build, mcp>=1.0.0 依存
   - `.mcp.json` — Claude Code接続設定（ローカル開発用）
2. **Claude Desktop設定スニペット** (`claude_desktop_config.json`)
3. **MCP Inspectorによる動作確認手順**

---

## Implementation Pattern

### Step 0: プロジェクト初期化

```bash
# パッケージ名を設定（スネークケース）
PKG=my_mcp_server
DIST=my-mcp-server

mkdir -p ${DIST}/src/${PKG} ${DIST}/tests
cd ${DIST}

# Python 3.11+ 仮想環境
python3 -m venv .venv
source .venv/bin/activate
pip install "mcp>=1.0.0" hatchling
```

### Step 1: pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-mcp-server"
version = "0.1.0"
description = "My MCP server description"
readme = "README.md"
license = { text = "MIT" }
requires-python = ">=3.11"
authors = [
    { name = "Your Name" },
]
keywords = ["mcp", "claude"]
dependencies = [
    "mcp>=1.0.0",
    # 追加依存: "paho-mqtt>=2.0.0", "pyyaml>=6.0", etc.
]

[project.scripts]
my-mcp-server = "my_mcp_server.server:run"

[tool.hatch.build.targets.wheel]
packages = ["src/my_mcp_server"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### Step 2: `__init__.py`

```python
# src/my_mcp_server/__init__.py
"""My MCP Server - brief description."""

__version__ = "0.1.0"
```

### Step 3: `server.py`（FastMCP基本形）

```python
# src/my_mcp_server/server.py
"""My MCP Server.

Tools:
- tool_a: Description of tool A
- tool_b: Description of tool B
"""

from __future__ import annotations

import json
import logging
from contextlib import asynccontextmanager
from typing import AsyncIterator

from mcp.server.fastmcp import FastMCP

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# ---- lifespan（起動/停止リソース管理）----
@asynccontextmanager
async def lifespan(server: FastMCP) -> AsyncIterator[dict]:
    """Initialize resources on startup, clean up on shutdown."""
    # 例: DB接続、バックグラウンドタスク、ネットワーク接続 等
    # resource = await SomeClient.connect()
    logger.info("My MCP server started")
    try:
        yield {}  # コンテキストデータ（mcp.get_context() で参照可能）
    finally:
        # await resource.close()
        logger.info("My MCP server stopped")


# ---- FastMCPインスタンス ----
mcp = FastMCP(
    "my-mcp-server",
    instructions="Describe what this MCP server does for the LLM",
    lifespan=lifespan,
)


# ---- ツール定義（型アノテーション → JSON Schema 自動生成）----

@mcp.tool()
def tool_a(param1: str, param2: int = 10) -> str:
    """Brief one-line description shown to LLM.

    Longer explanation if needed. Describe what the tool does,
    when to use it, and any important constraints.

    Args:
        param1: Description of param1.
        param2: Description of param2. Default: 10.
    """
    result = {
        "param1": param1,
        "param2": param2,
        "output": f"Processed: {param1}",
    }
    return json.dumps(result, indent=2, ensure_ascii=False)


@mcp.tool()
def tool_b(
    item_id: str,
    state: bool,
    priority: int = 10,
    duration_seconds: int | None = None,
) -> str:
    """Control an item's state.

    Args:
        item_id: Identifier of the item to control.
        state: True=ON/OPEN, False=OFF/CLOSE.
        priority: Priority level (1=highest, 30=lowest).
        duration_seconds: Auto-OFF timer. None means indefinite.
    """
    # 実装本体
    result = {
        "status": "ok",
        "item_id": item_id,
        "state": state,
        "priority": priority,
        "duration_seconds": duration_seconds,
    }
    return json.dumps(result, indent=2, ensure_ascii=False)


# ---- エントリーポイント ----

def run() -> None:
    """Entry point: stdio transport (default for Claude Desktop/Code)."""
    mcp.run(transport="stdio")


def run_http(host: str = "0.0.0.0", port: int = 8080) -> None:
    """Entry point: Streamable HTTP transport (for network clients)."""
    mcp.run(transport="streamable-http", host=host, port=port)


if __name__ == "__main__":
    run()
```

**ポイント**:
- `@mcp.tool()` のdocstringがLLMへのツール説明になる（丁寧に書け）
- 型アノテーションが自動でJSON Schemaに変換される（`int | None` → optional integer）
- 戻り値は `str`（JSON文字列）が推奨。LLMが解釈しやすい
- `ensure_ascii=False` で日本語を含む出力も正常に表示される

### Step 4: Streamable HTTP対応 (オプション)

```python
# pyproject.toml の [project.scripts] に追加:
# my-mcp-server-http = "my_mcp_server.server:run_http_main"

import argparse

def run_http_main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--host", default="0.0.0.0")
    parser.add_argument("--port", type=int, default=8080)
    args = parser.parse_args()
    run_http(host=args.host, port=args.port)
```

Streamable HTTP は `POST /mcp` でJSON-RPC + SSEレスポンス。
Ollamaなどネイティブにstdioをサポートしないクライアント向け（`mcp-proxy`等の中間ブリッジが必要な場合あり）。

### Step 5: テスト (`tests/test_tools.py`)

```python
"""Tests for MCP tools.

Note: MCP tools return JSON strings. Test by calling the functions directly.
MCP Inspector (npx @modelcontextprotocol/inspector) is used for protocol-level tests.
"""

import json
import pytest
from my_mcp_server.server import tool_a, tool_b


class TestToolA:
    def test_basic_call(self):
        result = json.loads(tool_a("hello"))
        assert result["param1"] == "hello"
        assert result["param2"] == 10  # default

    def test_custom_param2(self):
        result = json.loads(tool_a("world", param2=42))
        assert result["param2"] == 42

    def test_output_is_json_string(self):
        raw = tool_a("test")
        assert isinstance(raw, str)
        parsed = json.loads(raw)
        assert isinstance(parsed, dict)

    def test_japanese_output(self):
        result = json.loads(tool_a("日本語テスト"))
        assert "日本語テスト" in result["output"]


class TestToolB:
    def test_turn_on(self):
        result = json.loads(tool_b("item-001", True))
        assert result["status"] == "ok"
        assert result["state"] is True

    def test_turn_off(self):
        result = json.loads(tool_b("item-001", False))
        assert result["state"] is False

    def test_with_duration(self):
        result = json.loads(tool_b("item-001", True, duration_seconds=60))
        assert result["duration_seconds"] == 60

    def test_default_priority(self):
        result = json.loads(tool_b("item-001", True))
        assert result["priority"] == 10
```

**MCP Inspector（プロトコルレベルの動作確認）**:

```bash
# MCP Inspectorでサーバーを起動して対話テスト
npx @modelcontextprotocol/inspector .venv/bin/python -m my_mcp_server.server
# ブラウザで http://localhost:5173 を開く
# → list_tools, call_tool を手動で実行して確認
```

### Step 6: Claude Desktop / Claude Code 接続設定

**Claude Desktop** (`~/.config/claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "uvx",
      "args": ["my-mcp-server"],
      "env": {
        "MY_CONFIG_KEY": "value",
        "LOG_LEVEL": "INFO"
      }
    }
  }
}
```

**Claude Code** (プロジェクトルートの `.mcp.json`、ローカル開発用):

```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "/path/to/project/.venv/bin/python",
      "args": ["-m", "my_mcp_server.server"],
      "env": {
        "MY_CONFIG_KEY": "value"
      }
    }
  }
}
```

**uvxを使う場合（パッケージとして配布）**:

```bash
# ローカルインストールでテスト
pip install -e .
uvx --from . my-mcp-server  # ローカルwheelからuvxで起動

# PyPI公開後
uvx my-mcp-server  # インストール不要で起動
```

### Step 7: 動作確認チェックリスト

```bash
# 1. 単体テスト
pytest -v

# 2. ツール一覧確認（stdioプロトコル確認）
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | \
  .venv/bin/python -m my_mcp_server.server

# 3. ツール呼び出し確認
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"tool_a","arguments":{"param1":"test"}}}' | \
  .venv/bin/python -m my_mcp_server.server

# 4. MCP Inspector（ブラウザGUI）
npx @modelcontextprotocol/inspector .venv/bin/python -m my_mcp_server.server

# 5. Claude Code から接続（.mcp.json設置後）
# /mcp コマンドでサーバーリスト確認
```

---

## lifespan パターン（バックグラウンドタスクあり）

非同期バックグラウンドタスク（UDP受信、DB接続プール等）が必要な場合:

```python
import asyncio
from contextlib import asynccontextmanager
from mcp.server.fastmcp import FastMCP

# グローバル共有状態
_cache = {}
_bg_task: asyncio.Task | None = None


async def _background_worker():
    """永続バックグラウンドタスク（例: UDP受信ループ）"""
    while True:
        try:
            # データ取得処理
            data = await fetch_data()
            _cache.update(data)
            await asyncio.sleep(1)
        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error("Background worker error: %s", e)
            await asyncio.sleep(5)  # エラー後は少し待機


@asynccontextmanager
async def lifespan(server: FastMCP):
    global _bg_task
    _bg_task = asyncio.create_task(_background_worker())
    logger.info("Background worker started")
    try:
        yield {"cache": _cache}
    finally:
        _bg_task.cancel()
        try:
            await _bg_task
        except asyncio.CancelledError:
            pass
        logger.info("Background worker stopped")
```

---

## Notes / Limitations

- **FastMCP vs 低レベル Server**: `FastMCP` は `mcp.server.Server` の高レベルラッパー。プロダクション用途では `FastMCP` を推奨（実績あり、コード量大幅削減）
- **Python 3.11以上必須**: `mcp>=1.0.0` パッケージの要件。`int | None` 型構文も3.10+だが、実務上3.11+を前提にする
- **戻り値型**: `@mcp.tool()` の戻り値は `str`（JSON文字列）が最も安全。`dict`を返すことも技術的には可能だが、シリアライズ制御のため`json.dumps()`を明示推奨
- **環境変数**: 接続情報（ブローカーIP、APIキー等）は `os.environ.get()` で取得。`.mcp.json` の `env` フィールドで設定
- **Streamable HTTPのmcp-proxy**: Ollamaはネイティブにstdioをサポートしない。`mcp-proxy` 等の中間ブリッジが必要になる場合あり（要調査）
- **同期ツール関数**: `@mcp.tool()` は同期関数（`def`）でもOK。内部でスレッドプールで実行される。ただしブロッキングI/Oが多い場合は `async def` を使用
- **MCP Inspector**: `npx @modelcontextprotocol/inspector` はNode.js必須。`node --version` で確認
- **loggerは stderr へ**: MCPはstdout/stdinを通信に使うため、`logging` は必ずstderrへ出力される（デフォルト動作で問題なし）

---

**Skill Author**: 足軽2号提案（subtask_520）
**Last Updated**: 2026-02-20
