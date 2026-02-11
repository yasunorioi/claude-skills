# raspberrypi-os-installer-guide

**Category**: System Administration / Documentation
**Version**: 1.0.0
**Last Updated**: 2026-02-06

---

## 概要

Raspberry Pi系OSのインストール手順書を自動生成するスキル。

ユーザーが選択したOS（Home Assistant OS、Raspberry Pi OS、Ubuntu Server等）に対応した詳細なインストールガイドを生成し、初心者でも迷わず設定できるドキュメントを提供する。

### このスキルで出来ること

- **OSイメージのダウンロード手順**: 公式サイトからの取得方法、バージョン選択のガイダンス
- **SDカード書き込み手順**: Raspberry Pi Imager / Balena Etcher の使い方
- **初期設定ガイド**: SSH有効化、WiFi設定、初回起動時の注意点
- **トラブルシューティング**: よくある問題と対処法
- **OS別の特記事項**: 各OSに特有の設定やベストプラクティス

---

## 対応OS一覧

| OS名 | 用途 | 推奨ユーザー | 備考 |
|------|------|------------|------|
| **Home Assistant OS** | スマートホーム統合 | ホームオートメーション初心者〜上級者 | All-in-one、アドオンで機能拡張 |
| **Raspberry Pi OS（64-bit / 32-bit）** | 汎用デスクトップ / サーバー | 教育、プログラミング学習、汎用用途 | 旧名Raspbian、Debian系 |
| **Ubuntu Server for Raspberry Pi** | サーバー用途 | Linux経験者、クラウド連携 | LTS版推奨、apt-get管理 |
| **LibreELEC / OSMC** | メディアセンター | Kodi愛用者 | 専用メディアプレーヤーOS |
| **RetroPie** | ゲームエミュレーター | レトロゲーム愛好者 | エミュレータ統合環境 |
| **DietPi** | 軽量サーバー | 省電力運用、最小構成好き | RAM使用量最小化 |

---

## 共通手順テンプレート

全てのRaspberry Pi OSで共通する基本フローを以下に示す。

### フロー図

```
┌──────────────────────────────────────────────┐
│  Step 1: OSイメージのダウンロード              │
│  - 公式サイト / GitHub Releases              │
│  - バージョン選択（最新 or LTS）              │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Step 2: SDカード書き込み                     │
│  - Raspberry Pi Imager（推奨）               │
│  - Balena Etcher（代替）                     │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Step 3: 初期設定（書き込み後）               │
│  - SSH有効化（SSH空ファイル作成）             │
│  - WiFi設定（wpa_supplicant.conf）           │
│  - ユーザー設定（userconf.txt）              │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Step 4: Raspberry Piに挿入・起動             │
│  - SDカード挿入                              │
│  - 電源投入（5V/3A推奨）                     │
│  - 初回起動待機（3〜15分）                   │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Step 5: アクセス確認                         │
│  - SSH接続 / ブラウザアクセス                │
│  - 初期セットアップ実施                       │
└──────────────────────────────────────────────┘
```

---

## OS別詳細手順

### 1. Home Assistant OS

**用途**: スマートホーム統合プラットフォーム
**対象機種**: Raspberry Pi 4 / 5（推奨）、Raspberry Pi 3B+（動作可）

#### イメージダウンロード

```bash
# 公式サイト
https://www.home-assistant.io/installation/raspberrypi/

# Raspberry Pi 4用（64-bit推奨）
haos_rpi4-64-[version].img.xz

# または GitHub Releases
https://github.com/home-assistant/operating-system/releases
```

#### 推奨書き込みツール

**Raspberry Pi Imager**（最も簡単）

1. Raspberry Pi Imager を起動
2. 「OSを選ぶ」→「Other specific-purpose OS」→「Home assistants and home automation」→「Home Assistant」→「Raspberry Pi 4 / 400」
3. ストレージを選択（microSDカード）
4. 「書き込む」をクリック

#### 初回起動

- **待機時間**: 8〜15分（初回はパッケージダウンロードで時間がかかる）
- **アクセス方法**: `http://homeassistant.local:8123`
- **初期設定**: ブラウザでアカウント作成、ロケーション設定

#### 特記事項

- **SSH設定**: Phase 2で「SSH & Web Terminal」アドオンをインストール
- **WiFi設定**: OSレベルでのWiFi設定は非推奨（有線LAN推奨）
- **アドオン**: Mosquitto MQTT、File Editor、Samba等を追加インストール可能

---

### 2. Raspberry Pi OS（旧Raspbian）

**用途**: 汎用デスクトップ、プログラミング学習、サーバー
**対象機種**: Raspberry Pi 全シリーズ

#### イメージダウンロード

```bash
# 公式サイト
https://www.raspberrypi.com/software/operating-systems/

# バージョン選択
- Raspberry Pi OS with desktop（GUIあり、初心者推奨）
- Raspberry Pi OS Lite（CUI、サーバー用途）
- Raspberry Pi OS with desktop and recommended software（全部入り）

# 64-bit / 32-bit
Raspberry Pi 4 / 5 → 64-bit推奨
Raspberry Pi 3以前 → 32-bit推奨
```

#### 推奨書き込みツール

**Raspberry Pi Imager**

1. 「OSを選ぶ」→「Raspberry Pi OS (64-bit)」または「Raspberry Pi OS Lite (64-bit)」
2. ストレージを選択
3. **歯車アイコン（詳細設定）をクリック** → 重要！
   - ホスト名: `raspberrypi.local`（デフォルト）
   - **SSH有効化**: チェック
   - **ユーザー名とパスワード**: 設定（デフォルト: `pi` / `raspberry` は廃止されました）
   - **WiFi設定**: SSID、パスワード、国コード（JP）
   - **タイムゾーン**: `Asia/Tokyo`
4. 「保存」→「書き込む」

#### 初回起動

- **待機時間**: 3〜5分
- **SSH接続**: `ssh [ユーザー名]@raspberrypi.local`
- **デフォルトユーザー**: Imagerで設定したユーザー名（旧バージョンの `pi` は廃止）

#### 初期設定（SSH接続後）

```bash
# システムアップデート
sudo apt update && sudo apt upgrade -y

# タイムゾーン設定確認
timedatectl

# ロケール設定
sudo raspi-config
# → 5 Localisation Options → L1 Locale → ja_JP.UTF-8 選択

# VNC有効化（リモートデスクトップ）
sudo raspi-config
# → 3 Interface Options → I3 VNC → Yes
```

#### 特記事項

- **デフォルトユーザー変更**: 2022年4月以降、`pi` ユーザーは廃止。Imagerで設定必須
- **SSH有効化**: Imagerの詳細設定で有効化、または `/boot/ssh` 空ファイル作成
- **WiFi設定**: Imagerで事前設定可能、手動の場合は `wpa_supplicant.conf` を編集

---

### 3. Ubuntu Server for Raspberry Pi

**用途**: サーバー用途、クラウド連携、Kubernetes等
**対象機種**: Raspberry Pi 4 / 5（2GB以上推奨）

#### イメージダウンロード

```bash
# 公式サイト
https://ubuntu.com/download/raspberry-pi

# バージョン選択
- Ubuntu Server 24.04 LTS（64-bit）← 推奨（2029年までサポート）
- Ubuntu Server 22.04 LTS（64-bit）
```

#### 推奨書き込みツール

**Raspberry Pi Imager**

1. 「OSを選ぶ」→「Other general-purpose OS」→「Ubuntu」→「Ubuntu Server 24.04 LTS (64-bit)」
2. ストレージを選択
3. **歯車アイコン（詳細設定）** → 重要！
   - **SSH有効化**: パスワード認証を許可
   - **ユーザー名とパスワード**: `ubuntu` / 任意のパスワード
   - **WiFi設定**: SSID、パスワード
4. 「保存」→「書き込む」

#### 初回起動

- **待機時間**: 3〜5分
- **SSH接続**: `ssh ubuntu@ubuntu.local`（または IPアドレス）
- **初回ログイン**: パスワード変更を求められる

#### 初期設定（SSH接続後）

```bash
# システムアップデート
sudo apt update && sudo apt upgrade -y

# タイムゾーン設定
sudo timedatectl set-timezone Asia/Tokyo

# ホスト名変更（任意）
sudo hostnamectl set-hostname my-ubuntu-pi

# 再起動
sudo reboot
```

#### 特記事項

- **cloud-init**: 初回起動時に cloud-init が設定を適用（数分かかる）
- **デフォルトユーザー**: `ubuntu`（Imagerで変更可能）
- **パッケージ管理**: apt（Debian系）
- **Docker対応**: 公式リポジトリから簡単にインストール可能

---

### 4. LibreELEC / OSMC（メディアセンター）

**用途**: Kodi メディアプレーヤー専用OS
**対象機種**: Raspberry Pi 4 / 5 推奨

#### イメージダウンロード

**LibreELEC**:
```bash
https://libreelec.tv/downloads/
```

**OSMC**:
```bash
https://osmc.tv/download/
```

#### 推奨書き込みツール

- **LibreELEC**: 専用インストーラー（LibreELEC USB-SD Creator）
- **OSMC**: 専用インストーラー（OSMC Installer）

#### 特記事項

- Kodi 起動後、GUIで設定（SSH接続は設定メニューから有効化）
- メディアライブラリのスキャンに時間がかかる場合がある

---

### 5. RetroPie（ゲームエミュレーター）

**用途**: レトロゲームエミュレータ統合環境
**対象機種**: Raspberry Pi 4 / 5

#### イメージダウンロード

```bash
https://retropie.org.uk/download/
```

#### 推奨書き込みツール

**Raspberry Pi Imager** または **Balena Etcher**

#### 特記事項

- 初回起動時にコントローラー設定が必要
- ROMファイルの配置: `/home/pi/RetroPie/roms/[システム名]/`
- SSH有効化: デフォルトで有効（`pi` / `raspberry`）

---

## 初期設定の詳細

### SSH有効化（手動設定の場合）

Raspberry Pi Imagerの詳細設定を使わない場合、以下の方法でSSHを有効化できる。

#### 方法1: `/boot/ssh` 空ファイル作成

SDカードをPCに再挿入し、`/boot` パーティションに `ssh` という名前の空ファイルを作成する。

**Windows**:
```powershell
# /boot ドライブを開き、新しいテキストファイルを作成 → 拡張子を削除して "ssh" に名前変更
```

**macOS / Linux**:
```bash
touch /Volumes/boot/ssh
# または
touch /media/[ユーザー名]/boot/ssh
```

#### 方法2: Raspberry Pi Imagerの詳細設定

Imagerで書き込み前に「歯車アイコン」→「SSH有効化」にチェック。

---

### WiFi設定（手動設定の場合）

#### 方法1: `wpa_supplicant.conf` 作成

SDカードの `/boot` パーティションに `wpa_supplicant.conf` を作成。

**ファイル内容**:
```conf
country=JP
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="あなたのWiFi SSID"
    psk="あなたのWiFiパスワード"
    key_mgmt=WPA-PSK
}
```

**注意**:
- `country=JP` は日本の場合（US、UK等、国コードを正しく設定）
- パスワードは平文で記載（セキュリティ上、初回起動後に `/etc/wpa_supplicant/wpa_supplicant.conf` を確認し、暗号化推奨）

#### 方法2: Raspberry Pi Imagerの詳細設定

Imagerで書き込み前に「WiFi設定」でSSID、パスワード、国コードを入力。

---

### ユーザー設定（Raspberry Pi OS 2022年4月以降）

デフォルトユーザー `pi` が廃止されたため、初回起動前にユーザーを作成する。

#### 方法1: `userconf.txt` 作成（手動）

SDカードの `/boot` パーティションに `userconf.txt` を作成。

**ファイル内容**:
```
ユーザー名:暗号化パスワード
```

**暗号化パスワードの生成**:
```bash
# Linux / macOS
echo 'mypassword' | openssl passwd -6 -stdin

# 出力例: $6$rBoByrWRuby4fFes$6bHwTQ...（この文字列をコピー）
```

**userconf.txt の例**:
```
pi:$6$rBoByrWRuby4fFes$6bHwTQ...
```

#### 方法2: Raspberry Pi Imagerの詳細設定

Imagerで「ユーザー名とパスワード」を設定（最も簡単）。

---

## SDカード書き込みツールの詳細

### Raspberry Pi Imager（推奨）

**ダウンロード**: https://www.raspberrypi.com/software/

**特徴**:
- ✅ 公式ツール、最も確実
- ✅ OSイメージの自動ダウンロード機能
- ✅ 詳細設定（SSH、WiFi、ユーザー）の事前設定
- ✅ イメージ検証機能
- ✅ Windows / macOS / Linux 対応

**使い方**:
1. Imager起動 → 「OSを選ぶ」
2. 目的のOSを選択（または「Use custom」でローカルイメージ指定）
3. 「ストレージを選ぶ」→ microSDカード選択
4. **歯車アイコンクリック**（詳細設定）
   - SSH有効化
   - WiFi設定
   - ユーザー名・パスワード
   - タイムゾーン
5. 「書き込む」→ 完了

---

### Balena Etcher（代替）

**ダウンロード**: https://etcher.balena.io/

**特徴**:
- ✅ シンプルなUI
- ✅ 書き込み後の自動検証
- ✅ Windows / macOS / Linux 対応
- ❌ SSH/WiFi等の事前設定機能なし

**使い方**:
1. Etcher起動 → 「Flash from file」
2. ダウンロードしたイメージファイル（`.img` or `.img.xz`）を選択
3. 「Select target」→ microSDカード選択
4. 「Flash!」→ 完了

---

## トラブルシューティング

### 問題1: SDカードから起動しない

**症状**: 赤色LEDのみ点灯、緑色LEDが全く点滅しない

**原因と対処法**:

| 原因 | 対処法 |
|------|--------|
| イメージの書き込み失敗 | SDカードを再度書き込み |
| 不良SDカード | 別のSDカード（SanDisk Extreme等の高品質品）を試す |
| 電源不足 | 公式電源アダプタ（5V/3A）を使用、USBハブ経由の給電は避ける |
| 破損したイメージファイル | イメージを再ダウンロードして書き込み直し |
| SDカードの互換性問題 | Class 10以上、A1/A2規格のSDカードを使用 |

---

### 問題2: SSH接続できない

**症状**: `ssh: connect to host raspberrypi.local port 22: Connection refused`

**原因と対処法**:

| 原因 | 対処法 |
|------|--------|
| SSH未有効化 | `/boot/ssh` 空ファイル作成、またはImager詳細設定で有効化 |
| mDNS未対応（Windows） | Bonjour Services インストール、またはIPアドレスで直接接続 |
| WiFi接続失敗 | `wpa_supplicant.conf` の設定を確認、有線LAN接続で試す |
| ファイアウォール | ポート22を開放、またはファイアウォールを一時的に無効化 |
| ホスト名の重複 | `raspberrypi.local` が複数存在する場合、IPアドレスで接続 |

**IPアドレス確認方法**:
```bash
# ネットワークスキャン（nmap使用）
nmap -sn 192.168.1.0/24

# またはルーターの管理画面でDHCPリースを確認
```

---

### 問題3: WiFi接続できない

**症状**: SSH接続できない、または有線LANのみ動作

**原因と対処法**:

| 原因 | 対処法 |
|------|--------|
| `wpa_supplicant.conf` の記述ミス | SSIDとパスワードを再確認、国コード（`country=JP`）を確認 |
| 5GHz WiFi非対応 | Raspberry Pi 3B以前は2.4GHzのみ対応、ルーター設定を確認 |
| WiFiモジュール無効化 | `sudo rfkill unblock wifi` で有効化 |
| WiFiチップの問題 | USB WiFiアダプター使用、または有線LAN接続 |

---

### 問題4: `homeassistant.local` にアクセスできない

**症状**: ブラウザで `homeassistant.local:8123` を開いても接続できない

**原因と対処法**:

| 原因 | 対処法 |
|------|--------|
| mDNS未対応（Windows） | Bonjour Services インストール、またはIPアドレスで直接アクセス |
| 起動未完了 | 15〜20分待機してから再アクセス |
| LANケーブル未接続 | 物理的な接続を確認 |
| ポート8123がブロック | ファイアウォールでポート8123を開放 |

**mDNS確認方法**:
```bash
# macOS / Linux
ping homeassistant.local

# 応答があればOK、タイムアウトならIPアドレスで直接アクセス
```

---

### 問題5: 起動途中でフリーズする

**症状**: 緑LEDが点滅後、突然停止する、または無反応になる

**原因と対処法**:

| 原因 | 対処法 |
|------|--------|
| 電源不足 | USBデバイスを全て外す、電源アダプタを5V/3A（公式推奨）に交換 |
| SDカードの書き込み速度不足 | Class 10、A1/A2規格のSDカードを使用 |
| 過熱 | ヒートシンクまたは冷却ファンを追加 |
| メモリ不足 | Raspberry Pi 4 2GB → 4GB/8GBモデルに変更 |
| 破損したファイルシステム | SDカードを再度書き込み直し |

---

### 問題6: デフォルトユーザーでログインできない（Raspberry Pi OS）

**症状**: `pi` / `raspberry` でSSH接続できない

**原因と対処法**:

2022年4月以降、Raspberry Pi OSは `pi` ユーザーをデフォルトで作成しなくなった。

**対処法**:
1. Raspberry Pi Imagerの詳細設定でユーザー名・パスワードを設定
2. または `/boot/userconf.txt` を作成（上記「ユーザー設定」参照）

---

## ベストプラクティス

### SDカード選び

| 推奨度 | ブランド | モデル | 備考 |
|--------|---------|--------|------|
| ⭐⭐⭐ | SanDisk | Extreme 32GB/64GB | 高速・高耐久、A1/A2対応 |
| ⭐⭐⭐ | Samsung | EVO Plus 32GB/64GB | 高速、コスパ良 |
| ⭐⭐ | Kingston | Canvas Go! Plus | 低価格、A2対応 |
| ❌ | ノーブランド | - | 安価だが耐久性・速度に問題あり |

### 電源アダプタ選び

| 推奨度 | タイプ | 備考 |
|--------|--------|------|
| ⭐⭐⭐ | Raspberry Pi 公式電源アダプタ | 5V/3A、最も確実 |
| ⭐⭐ | Anker PowerPort III Nano | 高品質、USB PD対応 |
| ❌ | スマホ用充電器（5V/1A） | 電力不足でフリーズの原因 |

### セキュリティ設定

1. **デフォルトパスワード変更**: `pi` / `raspberry` のような弱いパスワードは使用禁止
2. **SSH鍵認証**: パスワード認証からSSH鍵認証に切り替え
3. **ファイアウォール**: `ufw` または `iptables` で不要なポートを閉じる
4. **自動アップデート**: `unattended-upgrades` で自動セキュリティパッチ適用

---

## 参考資料

### 公式ドキュメント

- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
- [Home Assistant - Raspberry Pi Installation](https://www.home-assistant.io/installation/raspberrypi/)
- [Ubuntu for Raspberry Pi](https://ubuntu.com/raspberry-pi)

### コミュニティ

- [Raspberry Pi Forums](https://forums.raspberrypi.com/)
- [Home Assistant Community](https://community.home-assistant.io/)
- [r/raspberry_pi (Reddit)](https://www.reddit.com/r/raspberry_pi/)

### ガイド

- [Installing Home Assistant on Raspberry Pi - Complete Guide](https://openelab.io/blogs/getting-started/installing-home-assistant-raspberry-pi)
- [Raspberry Pi OS Installation Guide - Pi My Life Up](https://pimylifeup.com/raspberry-pi-os/)

---

## スキル使用例

### 例1: Home Assistant OSインストールガイド生成

**入力**:
```
OS: Home Assistant OS
対象機種: Raspberry Pi 4 (4GB)
用途: スマートホーム制御
```

**出力**:
- イメージダウンロードURL
- Raspberry Pi Imager 使用手順
- 初回起動の待機時間（8〜15分）
- `http://homeassistant.local:8123` アクセス手順
- トラブルシューティング（mDNSアクセス不可、起動しない等）

---

### 例2: Raspberry Pi OS Lite（サーバー用途）インストールガイド生成

**入力**:
```
OS: Raspberry Pi OS Lite (64-bit)
対象機種: Raspberry Pi 4
用途: Webサーバー、Docker運用
```

**出力**:
- Raspberry Pi OS Lite イメージダウンロード
- SSH有効化（Imager詳細設定）
- WiFi設定（wpa_supplicant.conf）
- 初回SSH接続手順
- 初期設定（apt update、Docker インストール）

---

## まとめ

このスキルを使用することで、以下のメリットが得られる：

1. **初心者でも迷わない**: ステップバイステップの詳細手順
2. **OS別の特記事項を網羅**: Home Assistant OS、Raspberry Pi OS、Ubuntu Server等の違いを明確化
3. **トラブルシューティング完備**: よくある問題を事前に把握、対処法を即座に参照
4. **ベストプラクティス**: SDカード選び、電源選び、セキュリティ設定の推奨事項

**汎用性**: Raspberry Pi系OSのインストールが必要な全てのプロジェクトで利用可能。
