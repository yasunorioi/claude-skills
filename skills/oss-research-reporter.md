# oss-research-reporter

技術トピックについてWeb検索を行い、構造化された調査レポートをYAML/Markdown形式で自動生成するスキル。情報収集→整理→構造化出力を一貫して行う。

## メタデータ

| 項目 | 値 |
|------|-----|
| Skill ID | oss-research-reporter |
| Category | Research / Documentation |
| Version | 1.0.0 |
| Created | 2026-02-07 |
| Platform | Any（Web検索 + CLI） |

## Overview

技術調査タスクにおいて、以下のプロセスを体系化する：

1. 調査テーマの分解（サブトピック化）
2. 各サブトピックのWeb検索・情報収集
3. 情報の信頼性評価と整理
4. 構造化レポートの生成（YAML + Markdown）

主な特徴：
- 検索戦略の明示（どのキーワードで何を探すか）
- 情報源の信頼性ランク付け
- 構造化YAML出力（機械可読）とMarkdown出力（人間可読）の両方を生成
- 調査の再現性を確保（検索クエリ・情報源を記録）

## Use Cases

### 1. 技術選定のための調査

新しいフレームワーク/ライブラリ/ツールの導入前に、メリット・デメリット・代替案を体系的に調査。

### 2. OSSプロジェクトの技術的背景調査

OSSプロジェクト開発に先立ち、関連技術・既存実装・ベストプラクティスを調査。

### 3. アーキテクチャ設計のリサーチ

システム設計の意思決定に必要な技術的知見を収集・整理。

### 4. セキュリティ調査

脆弱性情報、ベストプラクティス、パッチ状況の調査。

## Skill Input

| パラメータ | 必須 | 説明 | デフォルト |
|-----------|------|------|-----------|
| `TOPIC` | Yes | 調査テーマ（例: `WireGuard VPN for IoT devices`） | - |
| `SUBTOPICS` | No | サブトピックリスト（自動分解も可） | 自動生成 |
| `DEPTH` | No | 調査の深さ: `shallow`/`standard`/`deep` | `standard` |
| `OUTPUT_FORMAT` | No | `yaml`, `markdown`, `both` | `both` |
| `MAX_SOURCES` | No | サブトピックあたりの最大情報源数 | `5` |
| `LANGUAGE` | No | 調査言語 | `en`（日本語ソースも含む） |
| `OUTPUT_DIR` | No | 出力先ディレクトリ | `./research/` |

## Generated Output

### ディレクトリ構造

```
./research/{topic_slug}/
├── report.yaml          # 構造化データ（機械可読）
├── report.md            # 人間可読レポート
└── sources.yaml         # 情報源一覧と信頼性評価
```

### YAML出力テンプレート

```yaml
# research/wireguard-iot/report.yaml
report:
  meta:
    topic: "WireGuard VPN for IoT devices"
    date: "2026-02-07"
    depth: "standard"
    author: "oss-research-reporter skill"

  executive_summary: |
    WireGuardはIoTデバイスのVPN接続に適した軽量プロトコルである。
    ...（3-5行の要約）

  subtopics:
    - id: 1
      title: "WireGuardの基本アーキテクチャ"
      findings:
        - point: "カーネルモジュールとして動作し、IPsec/OpenVPNより高速"
          source: "https://www.wireguard.com/papers/wireguard.pdf"
          confidence: high
        - point: "4,000行のコードベース（OpenVPN: 100,000行以上）"
          source: "https://www.wireguard.com/"
          confidence: high
      key_takeaway: "軽量・高速・シンプルでIoTに適する"

    - id: 2
      title: "IoTデバイスでの実装事例"
      findings:
        - point: "Raspberry PiでのWireGuard実装は公式サポート"
          source: "..."
          confidence: high
        - point: "ESP32での実装はwireguard-lwipライブラリで可能"
          source: "..."
          confidence: medium
      key_takeaway: "Linux系IoTデバイスでは容易、マイコンではライブラリ依存"

  conclusions:
    - "WireGuardはIoT VPNの最有力候補"
    - "リソース制約の厳しいマイコンではOpenVPNより優位"
    - "ただしUDP専用のため、TCPフォールバックが必要な環境には不向き"

  recommendations:
    - action: "Linux系IoTデバイスではWireGuardを標準採用"
      priority: high
    - action: "ESP32等のマイコンではwireguard-lwipの評価を実施"
      priority: medium
    - action: "企業ネットワークではTCPフォールバックの代替策を検討"
      priority: low
```

### Markdown出力テンプレート

```markdown
# 技術調査レポート: WireGuard VPN for IoT devices

**調査日**: 2026-02-07
**深度**: standard
**情報源数**: 15件

## エグゼクティブサマリー

WireGuardはIoTデバイスのVPN接続に適した軽量プロトコルである。...

## 1. WireGuardの基本アーキテクチャ

### 主要な知見
- カーネルモジュールとして動作し高速 [1]
- 4,000行のコードベース [2]

### キーテイクアウェイ
軽量・高速・シンプルでIoTに適する。

## 2. IoTデバイスでの実装事例
...

## 結論
...

## 推奨アクション
| 優先度 | アクション |
|--------|-----------|
| High   | Linux系IoTでWireGuard標準採用 |
| Medium | ESP32でのwireguard-lwip評価 |

## 情報源一覧
| # | URL | 信頼性 | 参照箇所 |
|---|-----|--------|----------|
| 1 | https://... | High | Section 1 |
| 2 | https://... | High | Section 1 |
```

## Implementation

### Phase 1: テーマの分解

```yaml
# 調査テーマをサブトピックに分解するテンプレート
decomposition:
  topic: "WireGuard VPN for IoT devices"
  subtopics:
    - id: 1
      title: "基本アーキテクチャと動作原理"
      search_queries:
        - "WireGuard architecture overview"
        - "WireGuard vs OpenVPN vs IPsec comparison"
    - id: 2
      title: "IoTデバイスでの実装事例"
      search_queries:
        - "WireGuard Raspberry Pi IoT"
        - "WireGuard embedded devices ESP32"
    - id: 3
      title: "セキュリティ特性"
      search_queries:
        - "WireGuard security audit results"
        - "WireGuard cryptographic primitives"
    - id: 4
      title: "パフォーマンスベンチマーク"
      search_queries:
        - "WireGuard throughput benchmark IoT"
        - "WireGuard CPU usage low power devices"
    - id: 5
      title: "制約と課題"
      search_queries:
        - "WireGuard limitations IoT"
        - "WireGuard UDP only NAT traversal"
```

### Phase 2: 情報収集（検索戦略）

```yaml
# 検索戦略テンプレート
search_strategy:
  primary_sources:
    - type: "公式ドキュメント"
      priority: 1
      example: "wireguard.com, docs.kernel.org"
    - type: "学術論文・技術レポート"
      priority: 2
      example: "arxiv.org, IEEE, ACM"
    - type: "公式ブログ・リリースノート"
      priority: 3
      example: "GitHub releases, project blogs"

  secondary_sources:
    - type: "技術ブログ"
      priority: 4
      example: "dev.to, medium.com, zenn.dev, qiita.com"
    - type: "Stack Overflow / GitHub Issues"
      priority: 5
      example: "stackoverflow.com, github.com/issues"

  search_tools:
    - "WebSearch（Claude Code内蔵）"
    - "gh search repos / gh api（GitHub CLI）"
    - "WebFetch（URL直接取得）"
```

### Phase 3: 信頼性評価

```yaml
# 情報源の信頼性ランク
confidence_levels:
  high:
    criteria:
      - "公式ドキュメント・論文"
      - "著名な開発者・組織のブログ"
      - "複数の独立した情報源で裏付け"
    label: "High"

  medium:
    criteria:
      - "技術ブログ（著者の実績あり）"
      - "Stack Overflow（高評価回答）"
      - "GitHub Issueの公式回答"
    label: "Medium"

  low:
    criteria:
      - "個人ブログ（裏付けなし）"
      - "古い情報（2年以上前）"
      - "単一情報源のみ"
    label: "Low"
```

### Phase 4: レポート生成

```bash
#!/bin/bash
# レポート生成の基本フロー

TOPIC="wireguard-iot"
OUTPUT_DIR="./research/${TOPIC}"
mkdir -p "${OUTPUT_DIR}"

# YAML出力
cat > "${OUTPUT_DIR}/report.yaml" << 'YAML_EOF'
report:
  meta:
    topic: "WireGuard VPN for IoT devices"
    date: "$(date +%Y-%m-%d)"
    depth: "standard"
  # ... (Phase 1-3の結果を構造化して書き込む)
YAML_EOF

# Markdown出力（YAMLから変換、または直接生成）
cat > "${OUTPUT_DIR}/report.md" << 'MD_EOF'
# 技術調査レポート: WireGuard VPN for IoT devices
# ... (人間可読形式で書き込む)
MD_EOF

echo "Report generated in: ${OUTPUT_DIR}/"
```

### 深度レベルの定義

| 深度 | サブトピック数 | 情報源/トピック | 所要時間目安 | 用途 |
|------|--------------|----------------|------------|------|
| `shallow` | 3-4 | 2-3 | 15-30分 | 概要把握、初期スクリーニング |
| `standard` | 5-7 | 3-5 | 30-60分 | 技術選定、設計判断 |
| `deep` | 8-12 | 5-10 | 1-2時間 | 詳細設計、論文調査 |

## 調査品質のチェックリスト

| チェック項目 | 説明 |
|-------------|------|
| 網羅性 | 主要な競合・代替技術に言及しているか |
| 最新性 | 情報源の日付が適切か（1年以内を推奨） |
| 多角性 | 賛成・反対の両方の意見を含んでいるか |
| 再現性 | 検索クエリと情報源が記録されているか |
| 実用性 | 具体的な推奨アクションが含まれているか |
| バイアス | 特定の技術・ベンダーに偏っていないか |

## 注意事項

- 調査結果は情報源の品質に依存する。必ず信頼性評価を付与すること
- 著作権に注意：引用は適切な範囲で行い、出典を明記すること
- 日本語と英語の両方で検索すると情報の幅が広がる
- 調査日を必ず記録すること（技術情報は陳腐化が早い）
- 「調べた結果、よくわからなかった」も重要な知見として記録すること

## 関連スキル

- `oss-competitive-analysis`: OSS競合プロジェクトの体系的比較
- `oss-competitor-analyzer`: OSS競合分析自動化（簡易版）
- `sequential-technical-guide-writer`: 既存手順書フォーマット踏襲の続編作成
