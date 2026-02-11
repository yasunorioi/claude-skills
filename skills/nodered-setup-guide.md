# Node-RED + Mosquitto セットアップガイド

Node-RED と Mosquitto MQTT Broker の環境構築ガイドを生成するスキル。
Docker Compose またはネイティブインストールの両方に対応。

## 使用方法

```
/nodered-setup-guide <環境タイプ> [オプション]
```

### パラメータ

| パラメータ | 説明 | 選択肢 |
|-----------|------|--------|
| 環境タイプ | インストール方法 | `docker` / `native` |
| --os | 対象OS（native時） | `ubuntu` / `debian` / `raspbian` / `macos` |
| --nodered-port | Node-REDポート | デフォルト: `1880` |
| --mqtt-port | MQTTポート | デフォルト: `1883` |
| --mqtt-ws-port | MQTT WebSocketポート | デフォルト: `9001` |
| --auth | 認証設定 | `none` / `basic` / `oauth` |
| --tls | TLS有効化 | `true` / `false` |
| --persistent | データ永続化 | `true` / `false` |

## 出力形式

1. **セットアップ手順書** - ステップバイステップのインストール手順
2. **docker-compose.yaml** - Docker環境用の構成ファイル
3. **mosquitto.conf** - Mosquitto設定ファイル
4. **settings.js** - Node-RED設定ファイル（認証設定含む）

## Docker環境サンプル

### docker-compose.yaml

```yaml
version: '3.8'

services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    networks:
      - iot-network

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - ./nodered/data:/data
    environment:
      - TZ=Asia/Tokyo
    depends_on:
      - mosquitto
    networks:
      - iot-network

networks:
  iot-network:
    driver: bridge
```

### mosquitto/config/mosquitto.conf

```conf
# Mosquitto Configuration
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_type all

# MQTT Listener
listener 1883
protocol mqtt

# WebSocket Listener
listener 9001
protocol websockets

# Authentication (basic auth enabled)
allow_anonymous false
password_file /mosquitto/config/passwd

# Access Control
acl_file /mosquitto/config/acl
```

### mosquitto/config/acl

```conf
# ACL Configuration
# user <username>
# topic [read|write|readwrite] <topic>

# Admin user - full access
user admin
topic readwrite #

# Sensor user - publish only to sensor topics
user sensor
topic write sensors/#

# Dashboard user - read all, write commands
user dashboard
topic read #
topic write commands/#
```

## ネイティブインストール手順（Ubuntu/Debian）

### 1. Mosquitto のインストール

```bash
# リポジトリ追加
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa

# パッケージ更新・インストール
sudo apt update
sudo apt install -y mosquitto mosquitto-clients

# 自動起動設定
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
```

### 2. Node-RED のインストール

```bash
# Node.js インストール（v18 LTS推奨）
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Node-RED インストール
sudo npm install -g --unsafe-perm node-red

# PM2でサービス化
sudo npm install -g pm2
pm2 start node-red
pm2 save
pm2 startup
```

### 3. 認証設定

```bash
# Mosquitto パスワードファイル作成
sudo mosquitto_passwd -c /etc/mosquitto/passwd admin

# Node-RED 認証設定（~/.node-red/settings.js を編集）
# adminAuth セクションを有効化
```

## Node-RED settings.js（認証有効）

```javascript
module.exports = {
    flowFile: 'flows.json',
    flowFilePretty: true,

    // Admin認証
    adminAuth: {
        type: "credentials",
        users: [{
            username: "admin",
            password: "$2b$08$xxxxxxxxxxxxx", // bcryptハッシュ
            permissions: "*"
        }]
    },

    // HTTPノード認証
    httpNodeAuth: {
        user: "user",
        pass: "$2b$08$xxxxxxxxxxxxx"
    },

    // UI設定
    uiPort: process.env.PORT || 1880,
    uiHost: "0.0.0.0",

    // ログ設定
    logging: {
        console: {
            level: "info",
            metrics: false,
            audit: false
        }
    },

    // MQTT接続設定（Mosquittoへの接続）
    functionGlobalContext: {
        mqttBroker: "mqtt://mosquitto:1883"
    }
};
```

## TLS設定（本番環境向け）

### Let's Encrypt証明書の取得

```bash
# certbot インストール
sudo apt install -y certbot

# 証明書取得
sudo certbot certonly --standalone -d your-domain.com

# Mosquitto TLS設定追加
cat >> /etc/mosquitto/conf.d/tls.conf << 'EOF'
listener 8883
cafile /etc/letsencrypt/live/your-domain.com/chain.pem
certfile /etc/letsencrypt/live/your-domain.com/cert.pem
keyfile /etc/letsencrypt/live/your-domain.com/privkey.pem
EOF
```

## 使用例

### IoTセンサーデータ収集環境

```bash
# 1. ディレクトリ構成作成
mkdir -p iot-platform/{mosquitto/{config,data,log},nodered/data}
cd iot-platform

# 2. 設定ファイル配置（上記サンプルを使用）

# 3. 起動
docker compose up -d

# 4. 動作確認
# Node-RED: http://localhost:1880
# MQTT: mosquitto_sub -h localhost -t '#' -v
```

### MQTT動作テスト

```bash
# ターミナル1: Subscribe
mosquitto_sub -h localhost -p 1883 -t 'sensors/#' -v

# ターミナル2: Publish
mosquitto_pub -h localhost -p 1883 -t 'sensors/temperature' -m '{"value": 25.5}'
```

## トラブルシューティング

| 症状 | 原因 | 対処法 |
|------|------|--------|
| 接続拒否 | ポート未開放 | `sudo ufw allow 1883` |
| 認証エラー | passwd未設定 | `mosquitto_passwd -c` |
| Node-RED起動しない | ポート競合 | `lsof -i :1880` で確認 |
| MQTT接続不安定 | キープアライブ | `keepalive_interval 60` |

## 注意事項

- 本番環境では必ずTLSと認証を有効化すること
- MQTTブローカーはインターネットに直接公開しないこと
- Node-REDのcredentialsファイルは暗号化されるが、バックアップ時は注意
- Docker環境ではボリュームのバックアップを忘れずに
