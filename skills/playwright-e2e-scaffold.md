# Playwright E2E Scaffold - Skill Definition

**Skill ID**: `playwright-e2e-scaffold`
**Category**: Testing / E2E
**Version**: 1.0.0
**Created**: 2026-02-11

---

## Overview

FastAPI + React (Vite) SPA のプロジェクトに Playwright E2E テスト基盤を一から構築するパターン。テスト用DBの分離、APIサーバーの自動起動、Page Objectパターン、認証済みfixtureを含む。

---

## Use Cases

- FastAPI + React SPA にE2Eテストを新規導入
- 既存のユニットテストに加えてE2Eテストを追加
- CI/CDパイプラインにE2Eテストを組み込む

---

## Skill Input

1. **プロジェクトパス**: 対象プロジェクトのルート
2. **技術スタック**: FastAPI + React (Vite) + SQLite（前提）
3. **認証方式**: JWT / セッション / なし
4. **主要画面**: テスト対象の画面一覧

---

## Skill Output

1. ディレクトリ構成
2. conftest.py（DB/サーバー/Playwright fixtures）
3. Page Objectクラス群
4. テストファイル群
5. pytest.ini / pyproject.toml 設定

---

## Implementation Pattern

### ディレクトリ構成

```
tests/
  e2e/
    __init__.py
    conftest.py              # DB初期化 + サーバー起動 + Playwright fixtures
    run_api_server.py        # テスト用APIサーバースクリプト
    pages/
      __init__.py            # Page Object re-exports
      base_page.py           # 共通操作（ナビゲーション、待機）
      dashboard_page.py      # ダッシュボード
      login_page.py          # ログイン
      fields_page.py         # ほ場管理（例）
    test_auth.py             # 認証テスト
    test_dashboard.py        # ダッシュボードテスト
    test_fields.py           # ほ場管理テスト（例）
```

### conftest.py — コア部分

```python
"""E2E テスト共通設定"""
import os, sys, time, shutil, signal, socket, subprocess, tempfile
from pathlib import Path
import pytest

PROJECT_ROOT = Path(__file__).parent.parent.parent

def _find_free_port() -> int:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind(("127.0.0.1", 0))
        return s.getsockname()[1]

def _wait_for_server(host: str, port: int, timeout: float = 30) -> bool:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        try:
            with socket.create_connection((host, port), timeout=1):
                return True
        except (ConnectionRefusedError, OSError):
            time.sleep(0.5)
    return False

def pytest_configure(config):
    config.addinivalue_line("markers", "e2e: End-to-End テスト")

@pytest.fixture(scope="session")
def test_db_path():
    """テスト用SQLite DBを作成し、スキーマ+シードデータを投入。"""
    tmpdir = tempfile.mkdtemp(prefix="e2e_")
    db_path = Path(tmpdir) / "e2e_test.db"

    sys.path.insert(0, str(PROJECT_ROOT))
    import your_app.db as db_mod  # プロジェクトに合わせて変更

    orig = db_mod.DB_PATH
    db_mod.DB_PATH = db_path
    db_mod.init_db()  # スキーマ適用 + 初期データ

    # 追加シードデータ（テスト用）
    import sqlite3
    conn = sqlite3.connect(str(db_path))
    # テスト用データINSERT
    conn.commit()
    conn.close()

    db_mod.DB_PATH = orig
    yield str(db_path)
    shutil.rmtree(tmpdir, ignore_errors=True)

@pytest.fixture(scope="session")
def api_port():
    return _find_free_port()

@pytest.fixture(scope="session")
def api_server(test_db_path, api_port):
    """E2Eテスト用FastAPIサーバーを起動。"""
    # フロントエンドビルド確認
    dist_dir = PROJECT_ROOT / "frontend" / "app" / "dist"
    if not (dist_dir / "index.html").exists():
        subprocess.run(
            ["npm", "run", "build"],
            cwd=str(PROJECT_ROOT / "frontend" / "app"),
            check=True, capture_output=True,
        )

    env = os.environ.copy()
    env["TEST_DB_PATH"] = test_db_path
    env["API_PORT"] = str(api_port)
    env["JWT_SECRET"] = "e2e_test_secret_key"

    proc = subprocess.Popen(
        [sys.executable, str(Path(__file__).parent / "run_api_server.py")],
        cwd=str(PROJECT_ROOT), env=env,
        stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    )

    if not _wait_for_server("127.0.0.1", api_port):
        proc.kill()
        stdout, stderr = proc.communicate(timeout=5)
        raise RuntimeError(f"Server failed: {stderr.decode()}")

    yield proc
    proc.send_signal(signal.SIGTERM)
    try:
        proc.wait(timeout=10)
    except subprocess.TimeoutExpired:
        proc.kill()

@pytest.fixture(scope="session")
def base_url(api_server, api_port):
    return f"http://127.0.0.1:{api_port}"

@pytest.fixture
def authenticated_page(page, base_url):
    """ログイン済みのPlaywright pageを提供。"""
    page.goto(f"{base_url}/login")
    page.get_by_label("ユーザー名").fill("admin")
    page.get_by_label("パスワード").fill("admin123")
    page.get_by_role("button", name="ログイン").click()
    page.wait_for_url(f"{base_url}/", timeout=10000)
    return page
```

### run_api_server.py

```python
"""E2Eテスト用サーバー起動スクリプト"""
import os, sys, uvicorn
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent.parent))

# テスト用DB差し替え
import your_app.db as db_mod
db_mod.DB_PATH = Path(os.environ["TEST_DB_PATH"])

port = int(os.environ.get("API_PORT", "8000"))
uvicorn.run("api.main:app", host="127.0.0.1", port=port)
```

### Page Object — base_page.py

```python
"""BasePage — 全ページ共通の操作"""
from playwright.sync_api import Page, expect

class BasePage:
    def __init__(self, page: Page, base_url: str = ""):
        self.page = page
        self.base_url = base_url

    def navigate_to(self, path: str) -> None:
        self.page.goto(f"{self.base_url}{path}")

    def wait_for_load(self, timeout: float = 10000) -> None:
        self.page.wait_for_load_state("networkidle", timeout=timeout)

    def get_toast_message(self) -> str:
        toast = self.page.locator(".toast, .notification, [role='alert']")
        if toast.count() > 0:
            return toast.first.text_content()
        return ""
```

### テストファイル例

```python
"""E2Eテスト: 認証"""
import pytest
from playwright.sync_api import Page, expect
from .pages import LoginPage, DashboardPage

@pytest.mark.e2e
class TestLogin:
    def test_successful_login(self, page: Page, base_url: str):
        login = LoginPage(page, base_url)
        login.goto()
        login.login("admin", "admin123")
        expect(page).to_have_url(f"{base_url}/")

    def test_wrong_password(self, page: Page, base_url: str):
        login = LoginPage(page, base_url)
        login.goto()
        login.login("admin", "wrong")
        login.expect_error_visible()
```

---

## 依存パッケージ

```bash
pip install pytest-playwright
playwright install chromium
```

### pyproject.toml 設定

```toml
[tool.pytest.ini_options]
markers = [
    "e2e: End-to-End tests",
    "slow: Slow tests",
]
```

---

## Best Practices

### 1. テスト用DB分離
- 本番DBに触れない。tmpdir に専用DB作成
- `session` scope で1回だけ初期化

### 2. Page Objectパターン
- ロケータはPage Objectに集約（テストコードにCSSセレクタを書かない）
- 実DOMを確認してからロケータを書く

### 3. テストの独立性
- 各テストは他テストの結果に依存しない
- テスト内でデータ作成→操作→検証→（クリーンアップ不要：session scope DB）

### 4. 待機戦略
- `wait_for_load_state("networkidle")` — API応答待ち
- `expect(locator).to_be_visible(timeout=...)` — 要素表示待ち
- `page.wait_for_timeout()` — 最終手段のみ（アニメーション等）

---

## Common Pitfalls

### 1. ロケータとDOMの不一致
**問題**: テストのCSSセレクタが実ReactコンポーネントのclassNameと一致しない
**解決**: `page.content()` やブラウザDevToolsで実DOMを確認

### 2. フロントエンドビルド忘れ
**問題**: `dist/` がないためサーバー起動しても画面が表示されない
**解決**: conftest.pyで自動ビルド（上記パターン参照）

### 3. ポート競合
**問題**: 固定ポートだと並列実行やローカル開発中に衝突
**解決**: `_find_free_port()` で動的ポート取得

### 4. ダイアログ（confirm/alert）未処理
**問題**: `window.confirm()` が出るとテストがハングする
**解決**: `page.on("dialog", lambda d: d.accept())` を事前登録

---

**Skill Author**: 足軽5号提案 / 将軍承認
**Last Updated**: 2026-02-11
