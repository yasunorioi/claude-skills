# WireGuard Peer Manager

> 統合済み: wireguard-client-config-generator + wireguard-peer-adder を統合

WireGuardピア管理の全手順を一元化するスキル。サーバー側ピア追加→クライアント設定生成→QRコード生成まで、エンドツーエンドで対応する。

## メタデータ

| 項目 | 値 |
|------|-----|
| Skill ID | wireguard-peer-manager |
| Category | Network / VPN |
| Version | 2.0.0 |
| Created | 2026-02-11 |
| Platform | Linux / macOS / Windows |

## Overview

WireGuardピア管理に関わる全プロセスを統合したスキル。サーバー管理者・クライアント管理者の双方に対応し、以下のシナリオをカバーする：

### カバーするシナリオ

1. **サーバー管理者モード（sudo必要）**: 新規ピアをサーバーに追加し、クライアント設定を生成する
2. **クライアント管理者モード（sudo不要）**: サーバー情報を元にクライアント設定のみを生成する
3. **一括追加モード**: 複数ピアを連続して追加する

### 主な機能

- 鍵生成（秘密鍵・公開鍵・事前共有鍵）
- サーバー側 wg0.conf への [Peer] セクション追記
- クライアント設定ファイル（.conf）の生成
- Full Tunnel / Split Tunnel の切り替え
- QRコード生成（スマートフォン用）
- WireGuardインターフェースのリロード

## Use Cases

### 1. 新しいリモートワーカーのVPN参加（サーバー管理者）

サーバーに新規ピアを追加し、クライアントに設定ファイルを配布する。

### 2. クライアント設定のみ生成（リモート）

サーバー管理者がサーバー公開鍵とエンドポイントを提供し、クライアント管理者が自分の設定ファイルを生成する。

### 3. Split Tunnel構成（リモートワーカー）

VPN経由で社内ネットワークのみアクセスし、一般インターネットは直接接続するSplit Tunnel設定。

### 4. IoTデバイスのVPN設定

Raspberry PiやPicoなどのIoTデバイス用の軽量VPN設定を生成。

### 5. スマートフォンのVPN設定

QRコード生成により、WireGuardモバイルアプリでの設定をワンスキャンで完了。

### 6. 複数ピアの一括追加

複数のクライアントを連続して追加する際、手作業ミスを防止。

## Skill Input

| パラメータ | 必須 | 説明 | デフォルト |
|-----------|------|------|-----------|
| **ピア情報** |
| `PEER_NAME` | Yes | ピアの識別名（例: `laptop-taro`, `phone-hanako`） | - |
| `CLIENT_ADDRESS` | Yes | クライアントに割り当てるVPN内IPアドレス（例: `10.0.0.10/32`） | - |
| **サーバー側操作（sudo必要）** |
| `MODE` | No | `server` or `client` | `server` |
| `SERVER_WG_CONF` | No | サーバー側WireGuard設定ファイルパス | `/etc/wireguard/wg0.conf` |
| **サーバー情報（client mode必須）** |
| `SERVER_PUBLIC_KEY` | Yes* | サーバーの公開鍵（client modeでは必須、server modeでは自動取得） | - |
| `SERVER_ENDPOINT` | Yes | サーバーのエンドポイント（例: `vpn.example.com:51820`） | - |
| **クライアント設定** |
| `CLIENT_PRIVATE_KEY` | No | クライアント秘密鍵（未指定時は自動生成） | 自動生成 |
| `DNS_SERVERS` | No | DNS サーバー | `1.1.1.1, 8.8.8.8` |
| `TUNNEL_MODE` | No | `full` or `split` | `full` |
| `ALLOWED_IPS` | No | ルーティング対象IP範囲 | `0.0.0.0/0, ::/0`（Full Tunnel） |
| `SPLIT_NETWORKS` | No | Split Tunnel時のネットワーク | `10.0.0.0/24, 192.168.0.0/16` |
| `MTU` | No | MTU値 | 未指定（WireGuardデフォルト） |
| `PERSISTENT_KEEPALIVE` | No | キープアライブ間隔（秒） | `25` |
| **出力設定** |
| `OUTPUT_DIR` | No | 出力先ディレクトリ | server: `/etc/wireguard/clients/`, client: `./wireguard-configs/` |
| `GENERATE_QR` | No | QRコード生成の有無 | `true` |

## Generated Output

### サーバー管理者モード（/etc/wireguard/clients/）

```
/etc/wireguard/clients/{PEER_NAME}/
├── privatekey                # クライアント秘密鍵（600）
├── publickey                 # クライアント公開鍵
├── presharedkey              # 事前共有鍵（600）
├── {PEER_NAME}.conf          # クライアント設定ファイル（600）
└── {PEER_NAME}.png           # QRコード画像
```

### クライアント管理者モード（./wireguard-configs/）

```
./wireguard-configs/
├── {PEER_NAME}.conf          # Full Tunnel設定
├── {PEER_NAME}-split.conf    # Split Tunnel設定（生成時のみ）
└── keys/
    └── {PEER_NAME}.key       # 秘密鍵（自動生成時、600）
```

### Full Tunnel設定ファイル（生成例）

```ini
[Interface]
# Profile: laptop-taro
# Generated: 2026-02-11
PrivateKey = <client_private_key>
Address = 10.0.0.10/32
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = <server_public_key>
PresharedKey = <preshared_key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### Split Tunnel設定ファイル（生成例）

```ini
[Interface]
# Profile: office-vpn (Split Tunnel)
# Generated: 2026-02-11
# Note: DNS is not set (split tunnel uses system DNS)
PrivateKey = <client_private_key>
Address = 10.0.0.10/32

[Peer]
PublicKey = <server_public_key>
PresharedKey = <preshared_key>
Endpoint = office-vpn.example.com:51820
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
PersistentKeepalive = 25
```

## Implementation

### 前提条件の確認

```bash
# WireGuardがインストールされているか
which wg || { echo "ERROR: wg not found. Install wireguard-tools"; exit 1; }

# qrencodeがインストールされているか（QRコード生成用、オプション）
which qrencode || { echo "WARNING: qrencode not found. QR generation will be skipped"; }

# サーバーモードの場合はroot権限を確認
if [ "${MODE}" = "server" ]; then
    [ "$(id -u)" -eq 0 ] || { echo "ERROR: Server mode requires sudo"; exit 1; }
fi
```

## サーバー管理者モード（sudo必要）

サーバーに新規ピアを追加し、クライアント設定を生成する完全な手順。

### STEP 1: 鍵の生成

```bash
PEER_NAME="laptop-taro"
OUTPUT_DIR="/etc/wireguard/clients/${PEER_NAME}"

# 出力ディレクトリ作成
mkdir -p "${OUTPUT_DIR}"
chmod 700 "${OUTPUT_DIR}"

# 秘密鍵・公開鍵・事前共有鍵の生成
wg genkey | tee "${OUTPUT_DIR}/privatekey" | wg pubkey > "${OUTPUT_DIR}/publickey"
wg genpsk > "${OUTPUT_DIR}/presharedkey"

# パーミッション設定（秘密鍵・事前共有鍵は所有者のみ読み取り可）
chmod 600 "${OUTPUT_DIR}/privatekey" "${OUTPUT_DIR}/presharedkey"

# 鍵を変数に格納
CLIENT_PRIVATE_KEY=$(cat "${OUTPUT_DIR}/privatekey")
CLIENT_PUBLIC_KEY=$(cat "${OUTPUT_DIR}/publickey")
PRESHARED_KEY=$(cat "${OUTPUT_DIR}/presharedkey")

echo "Client Public Key: ${CLIENT_PUBLIC_KEY}"
```

### STEP 2: サーバー設定に [Peer] を追記

```bash
SERVER_WG_CONF="/etc/wireguard/wg0.conf"
CLIENT_ADDRESS="10.0.0.10/32"

# バックアップ作成
cp "${SERVER_WG_CONF}" "${SERVER_WG_CONF}.bak.$(date +%Y%m%d%H%M%S)"

# [Peer] セクションを追記
cat >> "${SERVER_WG_CONF}" << EOF

# Peer: ${PEER_NAME} (added $(date +%Y-%m-%d))
[Peer]
PublicKey = ${CLIENT_PUBLIC_KEY}
PresharedKey = ${PRESHARED_KEY}
AllowedIPs = ${CLIENT_ADDRESS}
EOF

echo "Server config updated: ${SERVER_WG_CONF}"
```

### STEP 3: クライアント設定ファイルの生成

```bash
# サーバー公開鍵を取得
SERVER_PUBLIC_KEY=$(wg show wg0 public-key)
SERVER_ENDPOINT="vpn.example.com:51820"
DNS_SERVERS="1.1.1.1, 8.8.8.8"
ALLOWED_IPS="0.0.0.0/0, ::/0"

cat > "${OUTPUT_DIR}/${PEER_NAME}.conf" << EOF
[Interface]
# Peer: ${PEER_NAME}
PrivateKey = ${CLIENT_PRIVATE_KEY}
Address = ${CLIENT_ADDRESS}
DNS = ${DNS_SERVERS}

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
PresharedKey = ${PRESHARED_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = ${ALLOWED_IPS}
PersistentKeepalive = 25
EOF

chmod 600 "${OUTPUT_DIR}/${PEER_NAME}.conf"
echo "Client config created: ${OUTPUT_DIR}/${PEER_NAME}.conf"
```

### STEP 4: QRコード生成

```bash
if command -v qrencode &> /dev/null; then
    # PNG画像として保存
    qrencode -t png -o "${OUTPUT_DIR}/${PEER_NAME}.png" < "${OUTPUT_DIR}/${PEER_NAME}.conf"
    echo "QR code saved: ${OUTPUT_DIR}/${PEER_NAME}.png"

    # ターミナルにもQRコードを表示（UTF-8）
    qrencode -t ansiutf8 < "${OUTPUT_DIR}/${PEER_NAME}.conf"
else
    echo "WARNING: qrencode not installed. Skipping QR generation."
    echo "Install: sudo apt install qrencode"
fi
```

### STEP 5: WireGuardインターフェースのリロード

```bash
# 方法1: wg syncconf（ダウンタイムなし、推奨）
wg syncconf wg0 <(wg-quick strip wg0)
echo "WireGuard interface reloaded (syncconf)"

# 方法2: インターフェース再起動（syncconfが使えない場合）
# wg-quick down wg0 && wg-quick up wg0
# echo "WireGuard interface restarted"

# ピア一覧で追加を確認
wg show wg0 peers
```

### 一括スクリプト（サーバー管理者モード・全STEP統合）

```bash
#!/bin/bash
set -euo pipefail

# ===== 設定 =====
PEER_NAME="${1:?Usage: $0 <peer_name> <client_ip> [server_endpoint]}"
CLIENT_ADDRESS="${2:?Usage: $0 <peer_name> <client_ip> [server_endpoint]}"
SERVER_ENDPOINT="${3:-vpn.example.com:51820}"
SERVER_WG_CONF="/etc/wireguard/wg0.conf"
OUTPUT_DIR="/etc/wireguard/clients/${PEER_NAME}"
DNS_SERVERS="1.1.1.1, 8.8.8.8"
ALLOWED_IPS="0.0.0.0/0, ::/0"

# ===== 前提条件チェック =====
[ "$(id -u)" -eq 0 ] || { echo "ERROR: Run as root"; exit 1; }
command -v wg &> /dev/null || { echo "ERROR: wg not found"; exit 1; }

# サーバー公開鍵を取得
SERVER_PUBLIC_KEY=$(wg show wg0 public-key)

# ===== 鍵生成 =====
mkdir -p "${OUTPUT_DIR}" && chmod 700 "${OUTPUT_DIR}"
wg genkey | tee "${OUTPUT_DIR}/privatekey" | wg pubkey > "${OUTPUT_DIR}/publickey"
wg genpsk > "${OUTPUT_DIR}/presharedkey"
chmod 600 "${OUTPUT_DIR}/privatekey" "${OUTPUT_DIR}/presharedkey"

CLIENT_PRIVATE_KEY=$(cat "${OUTPUT_DIR}/privatekey")
CLIENT_PUBLIC_KEY=$(cat "${OUTPUT_DIR}/publickey")
PRESHARED_KEY=$(cat "${OUTPUT_DIR}/presharedkey")

# ===== サーバー設定追記 =====
cp "${SERVER_WG_CONF}" "${SERVER_WG_CONF}.bak.$(date +%Y%m%d%H%M%S)"
cat >> "${SERVER_WG_CONF}" << EOF

# Peer: ${PEER_NAME} (added $(date +%Y-%m-%d))
[Peer]
PublicKey = ${CLIENT_PUBLIC_KEY}
PresharedKey = ${PRESHARED_KEY}
AllowedIPs = ${CLIENT_ADDRESS}
EOF

# ===== クライアント設定生成 =====
cat > "${OUTPUT_DIR}/${PEER_NAME}.conf" << EOF
[Interface]
PrivateKey = ${CLIENT_PRIVATE_KEY}
Address = ${CLIENT_ADDRESS}
DNS = ${DNS_SERVERS}

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
PresharedKey = ${PRESHARED_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = ${ALLOWED_IPS}
PersistentKeepalive = 25
EOF
chmod 600 "${OUTPUT_DIR}/${PEER_NAME}.conf"

# ===== QRコード生成 =====
if command -v qrencode &> /dev/null; then
    qrencode -t png -o "${OUTPUT_DIR}/${PEER_NAME}.png" < "${OUTPUT_DIR}/${PEER_NAME}.conf"
    qrencode -t ansiutf8 < "${OUTPUT_DIR}/${PEER_NAME}.conf"
fi

# ===== リロード =====
wg syncconf wg0 <(wg-quick strip wg0)

echo ""
echo "===== Peer Added Successfully ====="
echo "Peer Name:    ${PEER_NAME}"
echo "Client IP:    ${CLIENT_ADDRESS}"
echo "Public Key:   ${CLIENT_PUBLIC_KEY}"
echo "Config File:  ${OUTPUT_DIR}/${PEER_NAME}.conf"
echo "===================================="
```

## クライアント管理者モード（sudo不要）

サーバー情報を元に、クライアント設定ファイルのみを生成する。

### 鍵の生成（自動生成時）

```bash
PROFILE_NAME="home-vpn"
OUTPUT_DIR="./wireguard-configs"
KEYS_DIR="${OUTPUT_DIR}/keys"

mkdir -p "${KEYS_DIR}"
chmod 700 "${KEYS_DIR}"

# wg コマンドによる鍵生成（sudo不要）
CLIENT_PRIVATE_KEY=$(wg genkey)
CLIENT_PUBLIC_KEY=$(echo "${CLIENT_PRIVATE_KEY}" | wg pubkey)

# 秘密鍵をファイルに保存
echo "${CLIENT_PRIVATE_KEY}" > "${KEYS_DIR}/${PROFILE_NAME}.key"
chmod 600 "${KEYS_DIR}/${PROFILE_NAME}.key"

echo "Client Public Key: ${CLIENT_PUBLIC_KEY}"
echo "(この公開鍵をサーバー管理者に渡してください)"
```

### Full Tunnel設定の生成

```bash
SERVER_PUBLIC_KEY="<server_public_key>"
SERVER_ENDPOINT="vpn.example.com:51820"
CLIENT_ADDRESS="10.0.0.10/32"
DNS_SERVERS="1.1.1.1, 8.8.8.8"

cat > "${OUTPUT_DIR}/${PROFILE_NAME}.conf" << EOF
[Interface]
# Profile: ${PROFILE_NAME}
# Generated: $(date +%Y-%m-%d)
PrivateKey = ${CLIENT_PRIVATE_KEY}
Address = ${CLIENT_ADDRESS}
DNS = ${DNS_SERVERS}

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF

chmod 600 "${OUTPUT_DIR}/${PROFILE_NAME}.conf"
echo "Config created: ${OUTPUT_DIR}/${PROFILE_NAME}.conf"
```

### Split Tunnel設定の生成

```bash
SPLIT_NETWORKS="10.0.0.0/24, 192.168.1.0/24"

cat > "${OUTPUT_DIR}/${PROFILE_NAME}-split.conf" << EOF
[Interface]
# Profile: ${PROFILE_NAME} (Split Tunnel)
# Generated: $(date +%Y-%m-%d)
# Note: DNS is not set (split tunnel uses system DNS)
PrivateKey = ${CLIENT_PRIVATE_KEY}
Address = ${CLIENT_ADDRESS}

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = ${SPLIT_NETWORKS}
PersistentKeepalive = 25
EOF

chmod 600 "${OUTPUT_DIR}/${PROFILE_NAME}-split.conf"
echo "Split tunnel config created: ${OUTPUT_DIR}/${PROFILE_NAME}-split.conf"
```

### MTU指定時の設定

```bash
# MTUを指定する場合（VPN over VPN、モバイル回線等で必要）
MTU=1380

cat > "${OUTPUT_DIR}/${PROFILE_NAME}.conf" << EOF
[Interface]
PrivateKey = ${CLIENT_PRIVATE_KEY}
Address = ${CLIENT_ADDRESS}
DNS = ${DNS_SERVERS}
MTU = ${MTU}

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF
```

### PresharedKey付き設定（より高いセキュリティ）

```bash
# 事前共有鍵の生成（sudo不要）
PRESHARED_KEY=$(wg genpsk)

cat > "${OUTPUT_DIR}/${PROFILE_NAME}.conf" << EOF
[Interface]
PrivateKey = ${CLIENT_PRIVATE_KEY}
Address = ${CLIENT_ADDRESS}
DNS = ${DNS_SERVERS}

[Peer]
PublicKey = ${SERVER_PUBLIC_KEY}
PresharedKey = ${PRESHARED_KEY}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF
```

### 一括生成スクリプト（複数プロファイル・クライアントモード）

```bash
#!/bin/bash
set -euo pipefail

# ===== 設定 =====
OUTPUT_DIR="./wireguard-configs"
KEYS_DIR="${OUTPUT_DIR}/keys"
mkdir -p "${KEYS_DIR}" && chmod 700 "${KEYS_DIR}"

# ===== プロファイル定義 =====
# 形式: "PROFILE_NAME|CLIENT_ADDRESS|SERVER_ENDPOINT|SERVER_PUBKEY|MODE|ALLOWED_IPS"
PROFILES=(
    "home-vpn|10.0.0.10/32|home.example.com:51820|<home_server_pubkey>|full|"
    "office-vpn|10.1.0.10/32|office.example.com:51820|<office_server_pubkey>|split|10.1.0.0/24,172.16.0.0/12"
    "cloud-vpn|10.2.0.10/32|cloud.example.com:51820|<cloud_server_pubkey>|full|"
)

for profile_def in "${PROFILES[@]}"; do
    IFS='|' read -r NAME ADDR ENDPOINT PUBKEY MODE NETWORKS <<< "${profile_def}"

    # 鍵生成
    PRIVKEY=$(wg genkey)
    PUBKEY_CLIENT=$(echo "${PRIVKEY}" | wg pubkey)
    echo "${PRIVKEY}" > "${KEYS_DIR}/${NAME}.key"
    chmod 600 "${KEYS_DIR}/${NAME}.key"

    # AllowedIPs決定
    if [ "${MODE}" = "split" ] && [ -n "${NETWORKS}" ]; then
        ALLOWED="${NETWORKS// /}"
        DNS_LINE=""
    else
        ALLOWED="0.0.0.0/0, ::/0"
        DNS_LINE="DNS = 1.1.1.1, 8.8.8.8"
    fi

    # 設定ファイル生成
    {
        echo "[Interface]"
        echo "# Profile: ${NAME} (${MODE} tunnel)"
        echo "# Generated: $(date +%Y-%m-%d)"
        echo "PrivateKey = ${PRIVKEY}"
        echo "Address = ${ADDR}"
        [ -n "${DNS_LINE}" ] && echo "${DNS_LINE}"
        echo ""
        echo "[Peer]"
        echo "PublicKey = ${PUBKEY}"
        echo "Endpoint = ${ENDPOINT}"
        echo "AllowedIPs = ${ALLOWED}"
        echo "PersistentKeepalive = 25"
    } > "${OUTPUT_DIR}/${NAME}.conf"
    chmod 600 "${OUTPUT_DIR}/${NAME}.conf"

    echo "[OK] ${NAME}: ${ADDR} (${MODE}) -> Public Key: ${PUBKEY_CLIENT}"
done

echo ""
echo "All configs generated in: ${OUTPUT_DIR}/"
echo "Share the public keys above with each server administrator."
```

## 各プラットフォームでの設定ファイル適用方法

### Linux

```bash
# 設定ファイルをコピー
sudo cp home-vpn.conf /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf

# 接続
sudo wg-quick up wg0

# 自動起動
sudo systemctl enable wg-quick@wg0
```

### macOS（WireGuard.app）

1. App Storeから WireGuard をインストール
2. アプリを起動 → 「Import tunnel(s) from file」
3. 生成した `.conf` ファイルを選択
4. 「Activate」で接続

### Windows（WireGuard.app）

1. wireguard.com から WireGuard をダウンロード・インストール
2. アプリを起動 → 「Import tunnel(s) from file」
3. 生成した `.conf` ファイルを選択
4. 「Activate」で接続

### iOS / Android（WireGuard App）

1. WireGuard アプリをインストール
2. `qrencode -t ansiutf8 < home-vpn.conf` でQRコードを表示
3. アプリの「+」→「Create from QR code」でスキャン

## セキュリティ注意事項

| 項目 | 対策 |
|------|------|
| 秘密鍵の保護 | `chmod 600` で所有者のみ読み取り可。パスワードマネージャーでの管理を推奨 |
| 出力ディレクトリ | `chmod 700`（所有者のみアクセス） |
| 設定ファイルの転送 | SCP、暗号化USBメモリ、パスワード付きZIPで転送。メール/Slack禁止 |
| バックアップの扱い | `.bak` ファイルにも秘密鍵が含まれる。不要になったら削除 |
| QRコードの扱い | 設定ファイル全文（秘密鍵含む）がエンコードされている。撮影後は画像を削除 |
| git追跡の防止 | `.gitignore` に `*.conf`, `*.key`, `wireguard-configs/`, `/etc/wireguard/` を追加 |
| 不要ファイルの削除 | 転送完了後、生成元マシンの設定ファイルを `shred -u` で安全削除 |
| PresharedKey | 量子コンピュータ耐性のため追加を推奨 |

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| `wg genkey` が見つからない | wireguard-tools 未インストール | `sudo apt install wireguard-tools` |
| `wg syncconf` がエラー | 設定ファイルの構文エラー | `wg-quick strip wg0` で確認 |
| 接続後インターネットに出られない | サーバー側でIP転送/NAT未設定 | サーバーの `PostUp`/`PostDown` iptables確認 |
| ピアが接続できない | AllowedIPsの重複 | 他ピアとのIP被りを確認 |
| ハンドシェイク失敗 | エンドポイント/ポートの問題 | `wg show wg0` でlatest handshakeを確認 |
| DNS解決できない | DNS設定の問題 | `resolvectl status` で確認。DNS行を修正 |
| Split Tunnel で社内のみ繋がらない | AllowedIPs が不足 | 社内ネットワークの全セグメントを追加 |
| MTU関連のタイムアウト | MTU値が大きすぎる | `MTU = 1280` から試す |
| QRコードが読めない | ターミナル幅不足 | PNG版を使用するか、ターミナルを広げる |

## 確認手順

### サーバー側確認（sudo必要）

```bash
# インターフェース状態確認
sudo wg show wg0

# ピア一覧確認
sudo wg show wg0 peers

# 特定ピアの詳細確認
sudo wg show wg0 dump | grep <client_public_key>

# ハンドシェイク確認（最終接続時刻）
sudo wg show wg0 latest-handshakes

# 転送量確認
sudo wg show wg0 transfer
```

### クライアント側確認

```bash
# 接続状態確認
sudo wg show

# ハンドシェイク確認
sudo wg show wg0 latest-handshakes

# 接続テスト（VPN内アドレスへのping）
ping 10.0.0.1

# DNSテスト
nslookup google.com

# ルーティング確認
ip route | grep wg0
```

## 関連スキル

- `iot-system-spec-generator`: IoTデバイスのネットワーク設計
- `shogun-zenn-writer`: VPN設定手順のドキュメント化

## 参考資料

- [WireGuard公式ドキュメント](https://www.wireguard.com/)
- [WireGuard Quick Start](https://www.wireguard.com/quickstart/)
- [wg(8) man page](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8)
