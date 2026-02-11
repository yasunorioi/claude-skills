# oss-competitive-analysis

> 統合済み: oss-competitor-analyzer, tech-comparison-reporter, iot-comm-comparison の内容を含む

特定ドメインのOSS競合プロジェクトを体系的に調査・分析し、比較表を作成するスキル。GitHub検索戦略と多軸評価で客観的な競合分析レポートを生成する。

## メタデータ

| 項目 | 値 |
|------|-----|
| Skill ID | oss-competitive-analysis |
| Category | Research / OSS Analysis |
| Version | 1.0.0 |
| Created | 2026-02-07 |
| Platform | Any（CLI + Web検索） |

## Overview

新規OSSプロジェクト立ち上げ時や技術選定時に、競合するOSSプロジェクトを体系的に調査・比較するスキル。以下を自動化する：

1. キーワード戦略に基づくGitHub検索
2. 候補プロジェクトの定量・定性評価
3. 多軸比較表の作成
4. SWOT分析と差別化ポイントの特定
5. 構造化レポートの出力

## Use Cases

### 1. 新規OSSプロジェクトの市場調査

自分のプロジェクトと似たOSSが既にないか、あるならどの程度のシェア・品質かを調査。

### 2. 技術選定の比較検討

同じ問題を解決する複数のOSSライブラリ/フレームワークを比較し、最適なものを選定。

### 3. 差別化戦略の立案

既存競合と自プロジェクトの違いを明確化し、READMEやドキュメントで訴求ポイントを整理。

## Skill Input

| パラメータ | 必須 | 説明 | デフォルト |
|-----------|------|------|-----------|
| `DOMAIN` | Yes | 調査対象ドメイン（例: `crop rotation planning`） | - |
| `KEYWORDS` | Yes | 検索キーワードリスト（例: `["crop rotation", "farm planning"]`） | - |
| `LANGUAGE_FILTER` | No | プログラミング言語フィルタ（例: `Python`） | なし（全言語） |
| `MIN_STARS` | No | 最低Star数フィルタ | `5` |
| `MAX_RESULTS` | No | 最大調査件数 | `20` |
| `OWN_PROJECT` | No | 自プロジェクトの概要（比較基準用） | なし |
| `OUTPUT_FORMAT` | No | `markdown` or `yaml` | `markdown` |

## Generated Output

### Markdown出力（デフォルト）

```markdown
# OSS競合分析レポート: {DOMAIN}

## 調査概要
- 調査日: 2026-02-07
- 検索キーワード: crop rotation, farm planning, agricultural management
- 対象言語: Python
- 発見プロジェクト数: 15件
- 詳細分析対象: 8件（Stars > 5）

## 比較表

| プロジェクト | Stars | Forks | 最終更新 | License | 言語 | 特徴 |
|-------------|-------|-------|----------|---------|------|------|
| project-a   | 1,234 | 156   | 2026-01  | MIT     | Python | 機能充実、大規模 |
| project-b   | 567   | 89    | 2025-12  | Apache  | JS     | Web UI、モバイル対応 |
| ...         | ...   | ...   | ...      | ...     | ...  | ... |

## 詳細分析
### project-a
- **リポジトリ**: https://github.com/xxx/project-a
- **概要**: ...
- **強み**: ...
- **弱み**: ...

## SWOT分析（自プロジェクト vs 競合）
...

## 推奨アクション
...
```

### YAML出力

```yaml
report:
  domain: "crop rotation planning"
  date: "2026-02-07"
  search_keywords: ["crop rotation", "farm planning"]
  competitors:
    - name: "project-a"
      url: "https://github.com/xxx/project-a"
      stars: 1234
      forks: 156
      last_update: "2026-01"
      license: "MIT"
      language: "Python"
      strengths: ["機能充実", "ドキュメント豊富"]
      weaknesses: ["UI古い", "依存関係多い"]
```

## Implementation

### Phase 1: 検索戦略の立案

```bash
# GitHub検索キーワードの組み合わせ生成

# 基本キーワード
PRIMARY_KEYWORDS=("crop rotation" "farm planning" "agricultural management")

# 技術キーワード（AND検索）
TECH_KEYWORDS=("python" "webapp" "sqlite" "gradio")

# 除外キーワード
EXCLUDE_KEYWORDS=("game" "tutorial" "homework")

# GitHub Search API クエリ構築例
# https://github.com/search?q=crop+rotation+language:python&type=repositories&s=stars&o=desc
```

### Phase 2: GitHub検索の実行

```bash
# gh CLI を使った検索（認証済み環境）
gh search repos "crop rotation" --language python --sort stars --limit 20 --json name,owner,stargazersCount,forksCount,updatedAt,licenseInfo,description

# 出力例をJSONで取得
gh search repos "crop rotation planning" \
    --sort stars \
    --limit 20 \
    --json name,owner,stargazersCount,forksCount,updatedAt,licenseInfo,description,url \
    > /tmp/search_results.json
```

### Phase 3: 候補の定量評価

```markdown
## 評価軸と重み付け

| 評価軸 | 重み | 説明 | 評価方法 |
|--------|------|------|----------|
| **人気度** | 20% | コミュニティの関心度 | Stars数、Forks数 |
| **活発度** | 25% | 開発の継続性 | 最終コミット日、コミット頻度、Issue応答時間 |
| **品質** | 20% | コードの成熟度 | テスト有無、CI/CD、ドキュメント |
| **機能性** | 20% | 機能の充実度 | README/docsの機能リスト |
| **エコシステム** | 15% | 周辺ツール・拡張性 | プラグイン、API、連携機能 |

### スコアリング基準

| 評価軸 | 5点 | 3点 | 1点 |
|--------|-----|-----|-----|
| Stars | > 1000 | 100-1000 | < 100 |
| 最終更新 | 1ヶ月以内 | 6ヶ月以内 | 6ヶ月以上 |
| テスト | CI/CD + 高カバレッジ | テストあり | テストなし |
| ドキュメント | 詳細 + 例 | README充実 | README最小限 |
| ライセンス | MIT/Apache | GPL | 不明/独自 |
```

### Phase 4: 定性分析テンプレート

```markdown
### {PROJECT_NAME} 詳細分析

**基本情報**
- リポジトリ: {URL}
- Stars: {N} / Forks: {N}
- ライセンス: {LICENSE}
- 主要言語: {LANGUAGE}
- 最終更新: {DATE}
- コントリビューター数: {N}

**機能概要**
- [ ] 機能A
- [ ] 機能B
- [ ] 機能C

**技術スタック**
- フロントエンド: {FE}
- バックエンド: {BE}
- データベース: {DB}
- デプロイ: {DEPLOY}

**強み（Strengths）**
1. ...
2. ...

**弱み（Weaknesses）**
1. ...
2. ...

**差別化ポイント（自プロジェクトとの比較）**
- 自プロジェクトにあって競合にないもの: ...
- 競合にあって自プロジェクトにないもの: ...
```

### Phase 5: SWOT分析テンプレート

```markdown
## SWOT分析（自プロジェクト vs 競合全体）

### Strengths（強み）
- 自プロジェクト固有の技術的優位性
- ターゲットユーザーへのフィット度

### Weaknesses（弱み）
- 競合と比べて不足している機能
- コミュニティ規模の差

### Opportunities（機会）
- 競合がカバーしていないニッチ
- 技術トレンドとの合致

### Threats（脅威）
- 大規模競合の存在
- 技術の陳腐化リスク
```

### Phase 6: 推奨アクションテンプレート

```markdown
## 推奨アクション

### 短期（1-2週間）
1. README に競合との差別化ポイントを明記
2. 競合にない独自機能をハイライト

### 中期（1-3ヶ月）
1. 競合の弱点を自プロジェクトの強みに転換
2. 不足機能のロードマップ作成

### 長期（3ヶ月以上）
1. コミュニティ形成（Discord/Discussions）
2. エコシステムの拡充（プラグイン/API）
```

## 検索のコツ

| テクニック | 説明 | 例 |
|-----------|------|-----|
| 同義語の網羅 | 同じ概念の異なる表現で検索 | `crop rotation` / `crop planning` / `farm rotation` |
| 技術スタック絞り込み | 特定言語・フレームワークに限定 | `language:python` |
| Star数フィルタ | ノイズ除去 | `stars:>10` |
| 更新日フィルタ | アクティブなプロジェクトのみ | `pushed:>2025-01-01` |
| トピック検索 | GitHubトピックタグで検索 | `topic:agriculture` |
| Awesome List確認 | キュレーションリストから発見 | `awesome-agriculture`, `awesome-farming` |

## 注意事項

- Stars数は人気度の一指標に過ぎない。低Stars でも高品質なプロジェクトは存在する
- ライセンスの互換性を必ず確認（GPL汚染に注意）
- フォーク元とフォーク先を混同しないこと
- アーカイブ済み（archived）リポジトリは除外すること
- 最終更新が古くても「完成している」場合がある（ライブラリ等）

## 統合元からの補足

### oss-competitor-analyzer: GitHub検索→YAML報告書の自動化パターン

**YAML報告書テンプレート**: OSS競合分析の結果を機械可読形式で出力するパターン。プログラマティックな処理が必要な場合に有用。

```yaml
meta:
  project: "{project_name}"
  keywords: {keywords}
  search_date: "{date}"
  total_found: {count}

competitors:
  - name: "{repo_name}"
    url: "https://github.com/{owner}/{repo}"
    metrics:
      stars: {stars}
      forks: {forks}
      last_commit: "{date}"
      contributors: {count}
      license: "{license}"
    evaluation:
      activity_score: {1-5}
      community_score: {1-5}
      maturity_score: {1-5}
      documentation_score: {1-5}
      total_score: {4-20}
    strengths: [...]
    weaknesses: [...]
    differentiation_notes: "..."

summary:
  landscape: |
    {市場全体の傾向}
  gaps: |
    {競合が埋めていないニーズ}
  recommendations: [...]
```

**スコアリングルール**: activity（更新頻度）、community（コミュニティ規模）、maturity（成熟度）、documentation（ドキュメント品質）の4軸で1〜5段階評価。

### tech-comparison-reporter: 技術比較表の作成テンプレート、評価基準フレームワーク

**重み付け比較表**: プロジェクト要件に応じて評価軸に重みを設定し、総合スコアで技術選定を客観化。

```markdown
| 比較軸 | 重み | 技術A | 技術B | 技術C |
|--------|------|-------|-------|-------|
| 機能性 | ×2 | ⭐⭐⭐⭐⭐ (10) | ⭐⭐⭐⭐ (8) | ⭐⭐⭐ (6) |
| パフォーマンス | ×1.5 | ⭐⭐⭐ (4.5) | ⭐⭐⭐⭐⭐ (7.5) | ⭐⭐⭐⭐ (6) |
| 学習コスト | ×1 | ⭐⭐⭐⭐ (4) | ⭐⭐⭐ (3) | ⭐⭐⭐⭐⭐ (5) |
| **合計** | | **32.5** | **30.5** | **27** |
```

**評価基準フレームワーク（5段階）**:
- ⭐⭐⭐⭐⭐ (5): 要件を完全に満たし追加のメリットがある
- ⭐⭐⭐⭐ (4): 要件を十分に満たす
- ⭐⭐⭐ (3): 要件は満たすが特筆すべき点はない
- ⭐⭐ (2): 一部要件を満たさない、または制約がある
- ⭐ (1): 要件を満たさない、または重大な問題がある

**推奨案の明確化**: 「場合によってAもBも」は意思決定の放棄。明確に1つを推奨し、次点を条件付きで示す。

### iot-comm-comparison: IoT通信方式（MQTT/HTTP/WebSocket等）の比較観点・評価軸

**IoT特有の評価軸**:
| 評価項目 | 説明 | 重視する場面 |
|---------|------|------------|
| 遅延 | レスポンスタイム、リアルタイム性 | 制御系、監視系 |
| 帯域効率 | ヘッダサイズ、ペイロード比率 | 低速回線、大量デバイス |
| 省電力性 | 通信頻度、接続維持コスト | バッテリー駆動センサー |
| 信頼性 | 到達保証、再送機構 | ミッションクリティカル |
| 双方向性 | サーバー→デバイス通信の容易さ | リモート設定、コマンド送信 |

**プロトコル別特性早見表**:
| プロトコル | 遅延 | 帯域効率 | 信頼性 | 双方向 | 実装難易度 |
|-----------|------|----------|--------|--------|-----------|
| MQTT | 低 | 高 | 高 | Yes | 低 |
| HTTP/REST | 中 | 低 | 高 | No | 最低 |
| WebSocket | 低 | 中 | 中 | Yes | 中 |
| TCP (Raw) | 最低 | 高 | 高 | Yes | 高 |
| UDP | 最低 | 高 | 低 | Yes | 高 |
| CoAP | 低 | 高 | 中 | Yes | 中 |

**複合システム推奨構成**: 単一プロトコルではなく、用途別に組み合わせる。
```
センサー → [MQTT] → ブローカー → [WebSocket] → ダッシュボード
                  ↓
            [HTTP/REST] → 設定API
                  ↓
制御器 ← [UDP/TCP] ← コントローラ
```

**通信量目安**: 1000デバイス、1分間隔での1時間あたり通信量
- MQTT: 約 6 MB（ヘッダ最小）
- HTTP: 約 48 MB（ヘッダ大）
- WebSocket: 約 7 MB（初期接続後軽量）
- UDP: 約 8 MB（固定長）

## 関連スキル

- `oss-research-reporter`: 技術調査レポート自動生成
