# git-confidential-docs-isolation

機密ドキュメントを別ブランチに隔離し、OSSリポジトリ公開やパブリック移行を安全に準備するスキル。

## 概要

プライベートリポジトリをOSS公開する際、機密情報（社内ドキュメント、顧客情報、アーキテクチャ詳細等）を含むディレクトリやファイルを別ブランチに隔離する。mainブランチからは削除し、.gitignoreで追跡除外することで、ローカルでは引き続き作業可能な状態を維持しつつ、リモートのmainブランチをクリーンに保つ。

## 使用方法

```
/git-confidential-docs-isolation [options]
```

### 入力パラメータ

| パラメータ | 必須 | 説明 | デフォルト |
|-----------|------|------|-----------|
| `--target` | Yes | 隔離対象のパス（ディレクトリまたはファイル） | - |
| `--branch` | No | 隔離用ブランチ名 | `private-docs` |
| `--base` | No | ベースブランチ名 | `main` |
| `--message` | No | コミットメッセージ | `Isolate confidential documents to {branch}` |

## ユースケース

### 1. OSSリポジトリ公開準備

プライベートで開発していたプロジェクトをOSS化する際、社内向けドキュメント（構成図、クライアント情報、内部設計書）を除去。

**例**: docs/internal/ を隔離して public リポジトリに移行

### 2. プライベート→パブリック移行

GitHub Privateリポジトリを Public に変更する前に、機密情報を含むフォルダを事前に隔離。

**例**: secrets/, credentials/, client_data/ を隔離

### 3. 社内ドキュメントの分離管理

OSS貢献者と社内メンバーで閲覧権限を分離。mainブランチはOSSとして公開し、private-docsブランチは社内専用。

**例**: business_plan.md, contracts/ を隔離

## Git操作手順

### STEP 1: 現在の状態確認

```bash
# 現在のブランチを確認
git branch

# 作業ディレクトリの状態確認
git status

# 隔離対象が存在することを確認
ls -la docs/
```

### STEP 2: 機密ファイル用ブランチ作成

```bash
# private-docs ブランチを作成（現在のmainから派生）
git checkout -b private-docs

# ブランチが作成されたことを確認
git branch
```

### STEP 3: 機密ファイルをprivate-docsにコミット

```bash
# 現在の状態で機密ファイルを含むコミットを作成
git add docs/
git commit -m "Preserve confidential documents in private-docs branch"

# コミットが作成されたことを確認
git log -1 --oneline
```

### STEP 4: mainブランチに戻り、機密ファイルを削除

```bash
# mainブランチに戻る
git checkout main

# 機密ファイル/ディレクトリを削除
git rm -r docs/

# 削除をコミット
git commit -m "Remove confidential docs from main branch (moved to private-docs)"
```

### STEP 5: .gitignoreに追加してローカルファイルを復元

```bash
# .gitignoreに追加（今後追跡しない）
echo "docs/" >> .gitignore
git add .gitignore
git commit -m "Add docs/ to .gitignore"

# private-docsブランチからローカルにファイルを復元（追跡されない）
git checkout private-docs -- docs/

# ローカルには存在するがgit管理外であることを確認
git status
# → docs/ は "Untracked files" として表示される
```

### STEP 6: 両ブランチをリモートにpush

```bash
# private-docsブランチをpush
git push -u origin private-docs

# mainブランチをpush
git checkout main
git push origin main
```

## 確認コマンド

### 各ブランチの内容確認

```bash
# mainブランチにdocs/が存在しないことを確認
git checkout main
ls -la docs/  # → "No such file or directory"
git ls-tree -r HEAD --name-only | grep docs/  # → 出力なし

# private-docsブランチにdocs/が存在することを確認
git checkout private-docs
ls -la docs/  # → ファイル一覧が表示される
git ls-tree -r HEAD --name-only | grep docs/  # → docs/配下のファイルが表示
```

### ローカルファイルの状態確認

```bash
# mainブランチに戻る
git checkout main

# ローカルにdocs/が存在することを確認
ls -la docs/  # → ファイル一覧が表示される（ローカルには残っている）

# gitの追跡状態を確認
git status
# → docs/は "Untracked files" として表示される（追跡されていない）
```

### リモートの状態確認

```bash
# 両ブランチがリモートに存在することを確認
git branch -r
# → origin/main
# → origin/private-docs

# GitHub上で確認
# main: docs/が存在しない
# private-docs: docs/が存在する
```

## ロールバック手順

### ケース1: STEP 4の後に間違いに気づいた場合

```bash
# mainブランチで削除コミットを取り消し
git checkout main
git reset --hard HEAD~1

# 作業が戻ったことを確認
git log -1 --oneline
ls -la docs/  # → docs/が復元されている
```

### ケース2: pushした後に間違いに気づいた場合

**⚠️ 注意**: リモートの履歴を変更するため、チーム開発では慎重に。

```bash
# ローカルで revert を作成（推奨）
git checkout main
git revert HEAD  # "Remove confidential docs" のコミットを打ち消す
git push origin main

# または、force pushで履歴を巻き戻す（非推奨・チームに影響大）
git reset --hard HEAD~1
git push --force origin main
```

### ケース3: private-docsブランチを削除してしまった場合

```bash
# reflogから復旧
git reflog
# → private-docsのコミットハッシュを見つける

git checkout -b private-docs <コミットハッシュ>
git push -u origin private-docs
```

## サンプル実行例

### 例1: docs/ ディレクトリを隔離

```bash
# STEP 1: 状態確認
$ git branch
* main

$ ls docs/
architecture.md  client_info.xlsx  internal_design.pdf

# STEP 2: private-docsブランチ作成
$ git checkout -b private-docs
Switched to a new branch 'private-docs'

# STEP 3: 現状をコミット
$ git add docs/
$ git commit -m "Preserve confidential documents in private-docs branch"
[private-docs a1b2c3d] Preserve confidential documents in private-docs branch
 3 files changed, 150 insertions(+)

# STEP 4: mainに戻って削除
$ git checkout main
$ git rm -r docs/
$ git commit -m "Remove confidential docs from main (moved to private-docs)"
[main d4e5f6g] Remove confidential docs from main (moved to private-docs)
 3 files changed, 150 deletions(-)

# STEP 5: .gitignore追加とローカル復元
$ echo "docs/" >> .gitignore
$ git add .gitignore
$ git commit -m "Add docs/ to .gitignore"

$ git checkout private-docs -- docs/
$ git status
On branch main
Untracked files:
  docs/

# STEP 6: push
$ git push -u origin private-docs
$ git push origin main
```

### 例2: 複数ファイルを個別に隔離

```bash
# secrets.yaml と credentials.json を隔離
$ git checkout -b private-secrets

$ git add secrets.yaml credentials.json
$ git commit -m "Preserve secrets in private-secrets branch"

$ git checkout main
$ git rm secrets.yaml credentials.json
$ git commit -m "Remove secrets from main (moved to private-secrets)"

$ echo "secrets.yaml" >> .gitignore
$ echo "credentials.json" >> .gitignore
$ git add .gitignore
$ git commit -m "Add secrets to .gitignore"

$ git checkout private-secrets -- secrets.yaml credentials.json
$ git push -u origin private-secrets
$ git push origin main
```

## 注意事項

### 1. Git履歴に残る機密情報の扱い

**⚠️ 重要**: この手順は「今後のmainブランチから機密ファイルを除外」するものであり、**過去のコミット履歴からは削除されない**。

**履歴から完全に削除する必要がある場合**:
```bash
# BFG Repo-Cleanerまたはgit filter-branchを使用
# ⚠️ 非常に破壊的な操作 - チーム全体に影響
git filter-branch --tree-filter 'rm -rf docs/' HEAD
git push --force origin main
```

**推奨アプローチ**:
- 新規リポジトリを作成し、クリーンな履歴で再スタート
- 古いリポジトリはアーカイブ（Read-Only）に設定

### 2. Force pushのリスク

- **チーム開発では原則禁止**: コラボレーターのローカル履歴と衝突
- Force pushが必要な場合は事前に全員に通知し、一斉に `git fetch && git reset --hard origin/main` を実行してもらう

### 3. コラボレーターへの影響

**push後のコラボレーターの対応**:
```bash
# 最新のmainとprivate-docsを取得
git fetch origin

# mainブランチを更新
git checkout main
git pull origin main

# private-docsブランチをチェックアウト
git checkout -b private-docs origin/private-docs

# mainに戻り、ローカルにdocs/を復元
git checkout main
git checkout private-docs -- docs/
```

### 4. GitHub Publicリポジトリ化の前に

- すべてのコミット履歴を手動レビュー: `git log --all --full-history -- docs/`
- 機密情報が過去のコミットに含まれていないか確認
- 必要に応じて新規リポジトリに移行（クリーンな履歴）

### 5. ブランチ保護設定

private-docsブランチを誤って削除しないよう、GitHub上でブランチ保護を設定：
- Settings → Branches → Add rule → `private-docs`
- "Require pull request reviews before merging" を有効化（オプション）

## 関連スキル

- `git-history-cleaner`: Git履歴から機密情報を完全削除（BFG Repo-Cleaner利用）
- `git-branch-strategy`: ブランチ戦略の設計（GitFlow, GitHub Flow等）
- `oss-release-checklist`: OSSリポジトリ公開前のチェックリスト

## 参考資料

- [GitHub Docs - Removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/)
- [Git Documentation - git-filter-branch](https://git-scm.com/docs/git-filter-branch)
