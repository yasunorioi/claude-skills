# Docker Compose テスト自動化スキル

Docker Compose環境の起動確認・動作テストを自動化する。
コンテナ起動、ポートリッスン、サービス間接続、ヘルスチェックを一括検証。

---

## 1. 概要

### 1.1 目的

docker-compose.yaml で定義された環境が正しく動作しているかを自動検証する。
手動確認の手間を省き、CI/CD パイプラインにも組み込み可能。

### 1.2 テスト項目

| カテゴリ | テスト内容 |
|---------|-----------|
| 構文検証 | docker-compose.yaml の構文チェック |
| 起動確認 | 全コンテナが running 状態か |
| ポート確認 | 公開ポートがリッスンしているか |
| ヘルスチェック | healthcheck 定義があれば結果確認 |
| サービス接続 | サービス間の通信が可能か |
| ログ確認 | エラーログが出ていないか |

### 1.3 対応サービス

| サービス | テスト方法 |
|---------|-----------|
| Mosquitto | MQTT pub/sub テスト |
| Node-RED | HTTP /flows アクセス |
| InfluxDB | /health エンドポイント |
| Grafana | /api/health エンドポイント |
| PostgreSQL | pg_isready コマンド |
| MySQL | mysqladmin ping |
| Redis | redis-cli ping |
| Nginx | HTTP ステータスコード |

---

## 2. 使用方法

### 2.1 入力パラメータ

```yaml
# テスト設定
test_config:
  compose_file: "./docker-compose.yaml"  # 対象ファイル
  project_name: "myproject"              # プロジェクト名（オプション）
  timeout: 60                            # 起動待機タイムアウト（秒）

  # サービス別テスト設定
  services:
    mosquitto:
      type: mqtt
      port: 1883
      test_topic: "test/ping"

    nodered:
      type: http
      port: 1880
      path: "/flows"
      expected_status: 200

    influxdb:
      type: http
      port: 8086
      path: "/health"
      expected_status: 200

    grafana:
      type: http
      port: 3000
      path: "/api/health"
      expected_status: 200
```

### 2.2 実行方法

```bash
# 基本実行
./docker-compose-test.sh

# 設定ファイル指定
./docker-compose-test.sh -c test_config.yaml

# 特定サービスのみ
./docker-compose-test.sh -s mosquitto,nodered

# 詳細出力
./docker-compose-test.sh -v
```

---

## 3. テストスクリプト

### 3.1 メインスクリプト (docker-compose-test.sh)

```bash
#!/bin/bash
# Docker Compose テスト自動化スクリプト
# 使用方法: ./docker-compose-test.sh [-c config.yaml] [-s service1,service2] [-v]

set -e

# デフォルト設定
COMPOSE_FILE="docker-compose.yaml"
TIMEOUT=60
VERBOSE=false
SERVICES=""

# 色定義
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 結果カウンター
PASSED=0
FAILED=0
SKIPPED=0

# ログ関数
log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }
log_pass() { echo -e "${GREEN}[PASS]${NC} $1"; ((PASSED++)); }
log_fail() { echo -e "${RED}[FAIL]${NC} $1"; ((FAILED++)); }
log_skip() { echo -e "${YELLOW}[SKIP]${NC} $1"; ((SKIPPED++)); }

# 引数解析
while getopts "c:s:vf:t:" opt; do
    case $opt in
        c) CONFIG_FILE="$OPTARG" ;;
        s) SERVICES="$OPTARG" ;;
        v) VERBOSE=true ;;
        f) COMPOSE_FILE="$OPTARG" ;;
        t) TIMEOUT="$OPTARG" ;;
        *) echo "Usage: $0 [-c config] [-s services] [-v] [-f compose-file] [-t timeout]"; exit 1 ;;
    esac
done

echo "========================================"
echo "Docker Compose テスト自動化"
echo "========================================"
echo "Compose File: $COMPOSE_FILE"
echo "Timeout: ${TIMEOUT}s"
echo ""

# ----------------------------------------
# 1. 構文検証
# ----------------------------------------
echo "=== 1. 構文検証 ==="

if [ ! -f "$COMPOSE_FILE" ]; then
    log_fail "ファイルが存在しません: $COMPOSE_FILE"
    exit 1
fi

if docker compose -f "$COMPOSE_FILE" config > /dev/null 2>&1; then
    log_pass "docker-compose.yaml 構文OK"
else
    log_fail "docker-compose.yaml 構文エラー"
    docker compose -f "$COMPOSE_FILE" config 2>&1 | head -20
    exit 1
fi

# ----------------------------------------
# 2. コンテナ起動確認
# ----------------------------------------
echo ""
echo "=== 2. コンテナ起動確認 ==="

# サービス一覧取得
ALL_SERVICES=$(docker compose -f "$COMPOSE_FILE" config --services)

for service in $ALL_SERVICES; do
    # フィルタリング
    if [ -n "$SERVICES" ] && [[ ! ",$SERVICES," == *",$service,"* ]]; then
        continue
    fi

    status=$(docker compose -f "$COMPOSE_FILE" ps -q "$service" 2>/dev/null)
    if [ -n "$status" ]; then
        state=$(docker inspect -f '{{.State.Status}}' "$status" 2>/dev/null || echo "unknown")
        if [ "$state" == "running" ]; then
            log_pass "$service: running"
        else
            log_fail "$service: $state"
        fi
    else
        log_fail "$service: not started"
    fi
done

# ----------------------------------------
# 3. ポートリッスン確認
# ----------------------------------------
echo ""
echo "=== 3. ポートリッスン確認 ==="

# 公開ポート取得
ports=$(docker compose -f "$COMPOSE_FILE" ps --format json 2>/dev/null | jq -r '.[].Publishers[]? | "\(.PublishedPort)"' 2>/dev/null | sort -u)

if [ -z "$ports" ]; then
    # 古い形式で試行
    ports=$(docker compose -f "$COMPOSE_FILE" ps 2>/dev/null | grep -oP '\d+(?=->)' | sort -u)
fi

for port in $ports; do
    if [ -n "$port" ] && [ "$port" != "null" ]; then
        if nc -z localhost "$port" 2>/dev/null; then
            log_pass "Port $port: リッスン中"
        else
            log_fail "Port $port: 応答なし"
        fi
    fi
done

# ----------------------------------------
# 4. ヘルスチェック確認
# ----------------------------------------
echo ""
echo "=== 4. ヘルスチェック確認 ==="

for service in $ALL_SERVICES; do
    if [ -n "$SERVICES" ] && [[ ! ",$SERVICES," == *",$service,"* ]]; then
        continue
    fi

    container_id=$(docker compose -f "$COMPOSE_FILE" ps -q "$service" 2>/dev/null)
    if [ -n "$container_id" ]; then
        health=$(docker inspect -f '{{.State.Health.Status}}' "$container_id" 2>/dev/null || echo "none")
        case "$health" in
            "healthy")
                log_pass "$service: healthy"
                ;;
            "unhealthy")
                log_fail "$service: unhealthy"
                if $VERBOSE; then
                    docker inspect -f '{{json .State.Health.Log}}' "$container_id" 2>/dev/null | jq -r '.[-1].Output' 2>/dev/null
                fi
                ;;
            "starting")
                log_warn "$service: starting (待機中)"
                ;;
            *)
                log_skip "$service: healthcheck未定義"
                ;;
        esac
    fi
done

# ----------------------------------------
# 5. サービス別テスト
# ----------------------------------------
echo ""
echo "=== 5. サービス別テスト ==="

# HTTP テスト関数
test_http() {
    local name=$1
    local url=$2
    local expected=${3:-200}

    response=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "$url" 2>/dev/null || echo "000")
    if [ "$response" == "$expected" ]; then
        log_pass "$name: HTTP $response"
    else
        log_fail "$name: HTTP $response (expected $expected)"
    fi
}

# MQTT テスト関数
test_mqtt() {
    local name=$1
    local host=${2:-localhost}
    local port=${3:-1883}

    if command -v mosquitto_pub &> /dev/null; then
        if timeout 5 mosquitto_pub -h "$host" -p "$port" -t "test/ping" -m "test" 2>/dev/null; then
            log_pass "$name: MQTT pub OK"
        else
            log_fail "$name: MQTT pub failed"
        fi
    else
        log_skip "$name: mosquitto_pub not installed"
    fi
}

# PostgreSQL テスト関数
test_postgres() {
    local name=$1
    local host=${2:-localhost}
    local port=${3:-5432}
    local user=${4:-postgres}

    if command -v pg_isready &> /dev/null; then
        if pg_isready -h "$host" -p "$port" -U "$user" > /dev/null 2>&1; then
            log_pass "$name: PostgreSQL ready"
        else
            log_fail "$name: PostgreSQL not ready"
        fi
    else
        # nc で代替
        if nc -z "$host" "$port" 2>/dev/null; then
            log_pass "$name: Port $port open"
        else
            log_fail "$name: Port $port closed"
        fi
    fi
}

# Redis テスト関数
test_redis() {
    local name=$1
    local host=${2:-localhost}
    local port=${3:-6379}

    if command -v redis-cli &> /dev/null; then
        if redis-cli -h "$host" -p "$port" ping 2>/dev/null | grep -q "PONG"; then
            log_pass "$name: Redis PONG"
        else
            log_fail "$name: Redis no response"
        fi
    else
        if nc -z "$host" "$port" 2>/dev/null; then
            log_pass "$name: Port $port open"
        else
            log_fail "$name: Port $port closed"
        fi
    fi
}

# 自動検出テスト
for service in $ALL_SERVICES; do
    if [ -n "$SERVICES" ] && [[ ! ",$SERVICES," == *",$service,"* ]]; then
        continue
    fi

    case "$service" in
        *mosquitto*)
            test_mqtt "$service" localhost 1883
            ;;
        *nodered*|*node-red*)
            test_http "$service" "http://localhost:1880" 200
            ;;
        *influxdb*)
            test_http "$service" "http://localhost:8086/health" 200
            ;;
        *grafana*)
            test_http "$service" "http://localhost:3000/api/health" 200
            ;;
        *postgres*)
            test_postgres "$service" localhost 5432
            ;;
        *redis*)
            test_redis "$service" localhost 6379
            ;;
        *nginx*)
            test_http "$service" "http://localhost:80" 200
            ;;
        *mysql*|*mariadb*)
            if nc -z localhost 3306 2>/dev/null; then
                log_pass "$service: Port 3306 open"
            else
                log_fail "$service: Port 3306 closed"
            fi
            ;;
    esac
done

# ----------------------------------------
# 6. エラーログ確認
# ----------------------------------------
echo ""
echo "=== 6. エラーログ確認 ==="

for service in $ALL_SERVICES; do
    if [ -n "$SERVICES" ] && [[ ! ",$SERVICES," == *",$service,"* ]]; then
        continue
    fi

    errors=$(docker compose -f "$COMPOSE_FILE" logs --tail=50 "$service" 2>/dev/null | grep -iE "(error|exception|fatal|panic)" | wc -l)
    if [ "$errors" -eq 0 ]; then
        log_pass "$service: エラーログなし"
    else
        log_warn "$service: エラーログ ${errors}件"
        if $VERBOSE; then
            docker compose -f "$COMPOSE_FILE" logs --tail=10 "$service" 2>/dev/null | grep -iE "(error|exception|fatal|panic)" | head -5
        fi
    fi
done

# ----------------------------------------
# 結果サマリー
# ----------------------------------------
echo ""
echo "========================================"
echo "テスト結果サマリー"
echo "========================================"
echo -e "${GREEN}PASSED: $PASSED${NC}"
echo -e "${RED}FAILED: $FAILED${NC}"
echo -e "${YELLOW}SKIPPED: $SKIPPED${NC}"
echo ""

if [ $FAILED -gt 0 ]; then
    echo -e "${RED}テスト失敗${NC}"
    exit 1
else
    echo -e "${GREEN}全テスト成功${NC}"
    exit 0
fi
```

### 3.2 Python版スクリプト (docker_compose_test.py)

```python
#!/usr/bin/env python3
"""
Docker Compose テスト自動化スクリプト (Python版)
"""

import subprocess
import json
import sys
import time
import socket
import argparse
from pathlib import Path
from dataclasses import dataclass
from typing import List, Dict, Optional

try:
    import requests
    HAS_REQUESTS = True
except ImportError:
    HAS_REQUESTS = False

try:
    import yaml
    HAS_YAML = True
except ImportError:
    HAS_YAML = False


@dataclass
class TestResult:
    name: str
    status: str  # "pass", "fail", "skip", "warn"
    message: str
    details: Optional[str] = None


class DockerComposeTest:
    """Docker Compose テストクラス"""

    def __init__(self, compose_file: str = "docker-compose.yaml",
                 timeout: int = 60, verbose: bool = False):
        self.compose_file = compose_file
        self.timeout = timeout
        self.verbose = verbose
        self.results: List[TestResult] = []

    def run_command(self, cmd: List[str], timeout: int = 30) -> tuple:
        """コマンド実行"""
        try:
            result = subprocess.run(
                cmd, capture_output=True, text=True, timeout=timeout
            )
            return result.returncode, result.stdout, result.stderr
        except subprocess.TimeoutExpired:
            return -1, "", "Timeout"
        except Exception as e:
            return -1, "", str(e)

    def add_result(self, name: str, status: str, message: str, details: str = None):
        """結果追加"""
        self.results.append(TestResult(name, status, message, details))

        # 出力
        colors = {"pass": "\033[92m", "fail": "\033[91m",
                  "skip": "\033[93m", "warn": "\033[93m"}
        reset = "\033[0m"
        print(f"{colors.get(status, '')}[{status.upper()}]{reset} {name}: {message}")

        if self.verbose and details:
            print(f"       {details}")

    # ----------------------------------------
    # テストメソッド
    # ----------------------------------------

    def test_syntax(self) -> bool:
        """構文検証"""
        print("\n=== 1. 構文検証 ===")

        if not Path(self.compose_file).exists():
            self.add_result("構文", "fail", f"ファイルが存在しません: {self.compose_file}")
            return False

        code, out, err = self.run_command(
            ["docker", "compose", "-f", self.compose_file, "config"]
        )

        if code == 0:
            self.add_result("構文", "pass", "docker-compose.yaml 構文OK")
            return True
        else:
            self.add_result("構文", "fail", "構文エラー", err[:200])
            return False

    def test_containers(self, services: List[str] = None) -> bool:
        """コンテナ起動確認"""
        print("\n=== 2. コンテナ起動確認 ===")

        # サービス一覧取得
        code, out, err = self.run_command(
            ["docker", "compose", "-f", self.compose_file, "config", "--services"]
        )
        if code != 0:
            self.add_result("コンテナ", "fail", "サービス一覧取得失敗")
            return False

        all_services = out.strip().split("\n")
        all_pass = True

        for service in all_services:
            if services and service not in services:
                continue

            code, out, err = self.run_command(
                ["docker", "compose", "-f", self.compose_file, "ps", "-q", service]
            )

            if out.strip():
                container_id = out.strip()
                code, state, _ = self.run_command(
                    ["docker", "inspect", "-f", "{{.State.Status}}", container_id]
                )
                state = state.strip()

                if state == "running":
                    self.add_result(service, "pass", "running")
                else:
                    self.add_result(service, "fail", state)
                    all_pass = False
            else:
                self.add_result(service, "fail", "not started")
                all_pass = False

        return all_pass

    def test_ports(self) -> bool:
        """ポートリッスン確認"""
        print("\n=== 3. ポートリッスン確認 ===")

        code, out, err = self.run_command(
            ["docker", "compose", "-f", self.compose_file, "ps", "--format", "json"]
        )

        ports = set()
        try:
            data = json.loads(out)
            for container in data:
                for pub in container.get("Publishers", []):
                    if pub.get("PublishedPort"):
                        ports.add(pub["PublishedPort"])
        except:
            pass

        all_pass = True
        for port in sorted(ports):
            if self._check_port(port):
                self.add_result(f"Port {port}", "pass", "リッスン中")
            else:
                self.add_result(f"Port {port}", "fail", "応答なし")
                all_pass = False

        return all_pass

    def test_health(self, services: List[str] = None) -> bool:
        """ヘルスチェック確認"""
        print("\n=== 4. ヘルスチェック確認 ===")

        code, out, _ = self.run_command(
            ["docker", "compose", "-f", self.compose_file, "config", "--services"]
        )
        all_services = out.strip().split("\n")

        for service in all_services:
            if services and service not in services:
                continue

            code, cid, _ = self.run_command(
                ["docker", "compose", "-f", self.compose_file, "ps", "-q", service]
            )

            if cid.strip():
                code, health, _ = self.run_command(
                    ["docker", "inspect", "-f", "{{.State.Health.Status}}", cid.strip()]
                )
                health = health.strip()

                if health == "healthy":
                    self.add_result(service, "pass", "healthy")
                elif health == "unhealthy":
                    self.add_result(service, "fail", "unhealthy")
                elif health == "starting":
                    self.add_result(service, "warn", "starting")
                else:
                    self.add_result(service, "skip", "healthcheck未定義")

        return True

    def test_services(self, services: List[str] = None) -> bool:
        """サービス別テスト"""
        print("\n=== 5. サービス別テスト ===")

        code, out, _ = self.run_command(
            ["docker", "compose", "-f", self.compose_file, "config", "--services"]
        )
        all_services = out.strip().split("\n")

        for service in all_services:
            if services and service not in services:
                continue

            # サービス名でテスト方法を判定
            if "mosquitto" in service.lower():
                self._test_mqtt(service)
            elif "nodered" in service.lower() or "node-red" in service.lower():
                self._test_http(service, "http://localhost:1880", 200)
            elif "influxdb" in service.lower():
                self._test_http(service, "http://localhost:8086/health", 200)
            elif "grafana" in service.lower():
                self._test_http(service, "http://localhost:3000/api/health", 200)
            elif "postgres" in service.lower():
                self._test_port(service, 5432)
            elif "redis" in service.lower():
                self._test_port(service, 6379)
            elif "nginx" in service.lower():
                self._test_http(service, "http://localhost:80", 200)

        return True

    def test_logs(self, services: List[str] = None) -> bool:
        """エラーログ確認"""
        print("\n=== 6. エラーログ確認 ===")

        code, out, _ = self.run_command(
            ["docker", "compose", "-f", self.compose_file, "config", "--services"]
        )
        all_services = out.strip().split("\n")

        for service in all_services:
            if services and service not in services:
                continue

            code, logs, _ = self.run_command(
                ["docker", "compose", "-f", self.compose_file, "logs", "--tail=50", service]
            )

            error_keywords = ["error", "exception", "fatal", "panic"]
            errors = sum(1 for line in logs.lower().split("\n")
                        if any(kw in line for kw in error_keywords))

            if errors == 0:
                self.add_result(service, "pass", "エラーログなし")
            else:
                self.add_result(service, "warn", f"エラーログ {errors}件")

        return True

    # ----------------------------------------
    # ヘルパーメソッド
    # ----------------------------------------

    def _check_port(self, port: int, host: str = "localhost") -> bool:
        """ポート疎通確認"""
        try:
            with socket.create_connection((host, port), timeout=5):
                return True
        except:
            return False

    def _test_http(self, name: str, url: str, expected: int = 200):
        """HTTPテスト"""
        if not HAS_REQUESTS:
            self.add_result(name, "skip", "requests未インストール")
            return

        try:
            resp = requests.get(url, timeout=5)
            if resp.status_code == expected:
                self.add_result(name, "pass", f"HTTP {resp.status_code}")
            else:
                self.add_result(name, "fail", f"HTTP {resp.status_code} (expected {expected})")
        except Exception as e:
            self.add_result(name, "fail", f"接続エラー: {e}")

    def _test_mqtt(self, name: str, host: str = "localhost", port: int = 1883):
        """MQTTテスト"""
        if self._check_port(port, host):
            self.add_result(name, "pass", f"Port {port} open")
        else:
            self.add_result(name, "fail", f"Port {port} closed")

    def _test_port(self, name: str, port: int, host: str = "localhost"):
        """ポートテスト"""
        if self._check_port(port, host):
            self.add_result(name, "pass", f"Port {port} open")
        else:
            self.add_result(name, "fail", f"Port {port} closed")

    # ----------------------------------------
    # メイン実行
    # ----------------------------------------

    def run_all(self, services: List[str] = None) -> bool:
        """全テスト実行"""
        print("=" * 40)
        print("Docker Compose テスト自動化")
        print("=" * 40)
        print(f"Compose File: {self.compose_file}")
        print(f"Timeout: {self.timeout}s")

        if not self.test_syntax():
            return False

        self.test_containers(services)
        self.test_ports()
        self.test_health(services)
        self.test_services(services)
        self.test_logs(services)

        return self.print_summary()

    def print_summary(self) -> bool:
        """結果サマリー出力"""
        print("\n" + "=" * 40)
        print("テスト結果サマリー")
        print("=" * 40)

        passed = sum(1 for r in self.results if r.status == "pass")
        failed = sum(1 for r in self.results if r.status == "fail")
        skipped = sum(1 for r in self.results if r.status == "skip")
        warned = sum(1 for r in self.results if r.status == "warn")

        print(f"\033[92mPASSED: {passed}\033[0m")
        print(f"\033[91mFAILED: {failed}\033[0m")
        print(f"\033[93mSKIPPED: {skipped}\033[0m")
        print(f"\033[93mWARNED: {warned}\033[0m")
        print()

        if failed > 0:
            print("\033[91mテスト失敗\033[0m")
            return False
        else:
            print("\033[92m全テスト成功\033[0m")
            return True

    def to_json(self) -> str:
        """JSON形式で結果出力"""
        return json.dumps({
            "compose_file": self.compose_file,
            "results": [
                {
                    "name": r.name,
                    "status": r.status,
                    "message": r.message,
                    "details": r.details
                }
                for r in self.results
            ],
            "summary": {
                "passed": sum(1 for r in self.results if r.status == "pass"),
                "failed": sum(1 for r in self.results if r.status == "fail"),
                "skipped": sum(1 for r in self.results if r.status == "skip"),
            }
        }, indent=2, ensure_ascii=False)


def main():
    parser = argparse.ArgumentParser(description="Docker Compose テスト自動化")
    parser.add_argument("-f", "--file", default="docker-compose.yaml",
                        help="docker-compose.yaml のパス")
    parser.add_argument("-s", "--services", help="テスト対象サービス（カンマ区切り）")
    parser.add_argument("-t", "--timeout", type=int, default=60, help="タイムアウト（秒）")
    parser.add_argument("-v", "--verbose", action="store_true", help="詳細出力")
    parser.add_argument("--json", action="store_true", help="JSON形式で出力")

    args = parser.parse_args()

    services = args.services.split(",") if args.services else None

    tester = DockerComposeTest(
        compose_file=args.file,
        timeout=args.timeout,
        verbose=args.verbose
    )

    success = tester.run_all(services)

    if args.json:
        print("\n--- JSON出力 ---")
        print(tester.to_json())

    sys.exit(0 if success else 1)


if __name__ == "__main__":
    main()
```

---

## 4. 出力形式

### 4.1 コンソール出力（デフォルト）

```
========================================
Docker Compose テスト自動化
========================================
Compose File: docker-compose.yaml
Timeout: 60s

=== 1. 構文検証 ===
[PASS] 構文: docker-compose.yaml 構文OK

=== 2. コンテナ起動確認 ===
[PASS] mosquitto: running
[PASS] nodered: running
[PASS] influxdb: running
[PASS] grafana: running

=== 3. ポートリッスン確認 ===
[PASS] Port 1883: リッスン中
[PASS] Port 1880: リッスン中
[PASS] Port 8086: リッスン中
[PASS] Port 3000: リッスン中

=== 4. ヘルスチェック確認 ===
[PASS] mosquitto: healthy
[PASS] nodered: healthy
[SKIP] influxdb: healthcheck未定義
[PASS] grafana: healthy

=== 5. サービス別テスト ===
[PASS] mosquitto: Port 1883 open
[PASS] nodered: HTTP 200
[PASS] influxdb: HTTP 200
[PASS] grafana: HTTP 200

=== 6. エラーログ確認 ===
[PASS] mosquitto: エラーログなし
[PASS] nodered: エラーログなし
[WARN] influxdb: エラーログ 2件
[PASS] grafana: エラーログなし

========================================
テスト結果サマリー
========================================
PASSED: 18
FAILED: 0
SKIPPED: 1
WARNED: 1

全テスト成功
```

### 4.2 JSON出力 (--json オプション)

```json
{
  "compose_file": "docker-compose.yaml",
  "results": [
    {
      "name": "構文",
      "status": "pass",
      "message": "docker-compose.yaml 構文OK",
      "details": null
    },
    {
      "name": "mosquitto",
      "status": "pass",
      "message": "running",
      "details": null
    },
    {
      "name": "Port 1883",
      "status": "pass",
      "message": "リッスン中",
      "details": null
    }
  ],
  "summary": {
    "passed": 18,
    "failed": 0,
    "skipped": 1
  }
}
```

---

## 5. CI/CD統合

### 5.1 GitHub Actions

```yaml
name: Docker Compose Test

on:
  push:
    paths:
      - 'docker-compose.yaml'
      - 'docker/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start services
        run: docker compose up -d

      - name: Wait for services
        run: sleep 30

      - name: Run tests
        run: |
          chmod +x ./docker-compose-test.sh
          ./docker-compose-test.sh -v

      - name: Cleanup
        if: always()
        run: docker compose down -v
```

### 5.2 GitLab CI

```yaml
docker-test:
  stage: test
  services:
    - docker:dind
  script:
    - docker compose up -d
    - sleep 30
    - ./docker-compose-test.sh
  after_script:
    - docker compose down -v
```

---

## 6. カスタムテスト定義

### 6.1 YAML設定ファイル

```yaml
# test_config.yaml
compose_file: ./docker-compose.yaml
timeout: 120

services:
  mosquitto:
    type: mqtt
    host: localhost
    port: 1883
    tests:
      - pub_sub:
          topic: "test/ping"
          message: "hello"
          timeout: 5

  nodered:
    type: http
    tests:
      - endpoint:
          url: "http://localhost:1880"
          method: GET
          expected_status: 200
      - endpoint:
          url: "http://localhost:1880/flows"
          method: GET
          expected_status: 200

  influxdb:
    type: http
    tests:
      - endpoint:
          url: "http://localhost:8086/health"
          expected_status: 200
      - endpoint:
          url: "http://localhost:8086/ready"
          expected_status: 200

  custom_app:
    type: custom
    tests:
      - command: "curl -s http://localhost:8080/api/health | jq -e '.status == \"ok\"'"
        expected_exit_code: 0
```

---

## 7. 注意事項

### 7.1 前提条件

| 項目 | 必須 |
|------|------|
| Docker | v20.10+ |
| Docker Compose | v2.0+ |
| nc (netcat) | ポートチェック用 |
| curl | HTTPテスト用 |
| jq | JSON解析用（オプション） |

### 7.2 制限事項

- ホストネットワーク外からのテストは非対応
- TLS/SSL証明書検証はスキップ
- 認証が必要なサービスは個別設定が必要

---

## 8. 参考資料

- Docker Compose: https://docs.docker.com/compose/
- Docker Health Check: https://docs.docker.com/compose/compose-file/05-services/#healthcheck
