# Quarto Markdown Publishing CI/CD Specification

**Version**: 1.0
**Last Updated**: 2026-01-03
**Status**: Design Phase

---

## 目次

1. [システム概要](#システム概要)
2. [ディレクトリ構造](#ディレクトリ構造)
3. [ブランチ戦略](#ブランチ戦略)
4. [CI/CDワークフロー設計](#cicdワークフロー設計)
5. [GitHub Branch Protection Rules](#github-branch-protection-rules)
6. [運用フロー](#運用フロー)
7. [セキュリティ考慮事項](#セキュリティ考慮事項)
8. [オプション機能](#オプション機能)

---

## システム概要

### 目的

`raws/` ディレクトリに追加されたMarkdownファイル（`.md`、`.qmd`）を、Quartoを使用して自動的にHTMLに変換し、GitHub Pagesで公開するCI/CDシステムを構築する。

### 主要コンポーネント

1. **Quartoレンダリングエンジン** - Markdown → HTML変換
2. **GitHub Actions CI/CD** - 自動ビルド・デプロイパイプライン
3. **GitHub Pages** - 静的サイトホスティング
4. **ブランチ保護機構** - mainブランチの健全性保証

### 設計原則

- **安全性優先**: mainブランチは常に正常な状態を維持
- **早期検出**: エラーはマージ前に検出
- **Public Repository対応**: 機密情報の漏洩防止
- **ロバスト性**: レンダリング失敗時の適切なハンドリング

---

## ディレクトリ構造

```
blog/
├── raws/                           # Markdownソースファイル格納
│   ├── index.md                   # サイトトップページ
│   ├── article-001.md             # 個別記事
│   ├── article-002.qmd            # Quarto Markdown記事
│   └── category/                  # サブディレクトリもサポート
│       └── sub-article.md
│
├── _quarto.yml                     # Quarto設定ファイル
├── _site/                          # レンダリング後HTML出力（gitignore）
├── .quarto/                        # Quartoキャッシュ（gitignore）
│
├── .github/
│   └── workflows/
│       ├── validate-content.yml   # Validation CI
│       └── deploy-pages.yml       # Deploy CI
│
├── .gitignore
├── SPEC.md                         # 本仕様書
├── PLAN.md                         # 実装計画
└── README.md
```

### ディレクトリ役割

- **`raws/`**: Markdownソースの唯一の格納場所
- **`_site/`**: Quartoが生成するHTML出力（Git管理外）
- **`.quarto/`**: Quartoキャッシュディレクトリ（Git管理外）

---

## ブランチ戦略

### ブランチ構成

```
main (保護ブランチ)
  ├── update/add-new-article      # 記事追加
  ├── update/modify-config        # 設定変更
  └── fix/typo-correction         # 修正
```

### ブランチ命名規則

| Prefix | 用途 | 例 |
|--------|------|-----|
| `update/*` | 記事追加・更新 | `update/add-quarto-tutorial` |
| `fix/*` | 修正（typo、リンク切れ等） | `fix/broken-link-in-post1` |
| `config/*` | 設定変更 | `config/change-theme` |

### mainブランチ保護方針

- ✅ **常に正常な状態を維持**
- ❌ **直接pushを禁止**
- ✅ **Pull Requestによるマージ必須**
- ✅ **Validation CI通過が必須**

---

## CI/CDワークフロー設計

### 全体フロー

```
┌─────────────────────────────────────────────────────────┐
│  開発者: update/* ブランチで記事作成                      │
└─────────────────────┬───────────────────────────────────┘
                      │ git push
                      ▼
┌─────────────────────────────────────────────────────────┐
│  GitHub Actions: Validation CI 実行                      │
│  - Quartoレンダリング検証                                 │
│  - リンクチェック（オプション）                            │
│  - Markdownリント（オプション）                           │
└─────────────────────┬───────────────────────────────────┘
                      │
            ┌─────────┴─────────┐
            │                   │
         ❌ FAIL            ✅ PASS
            │                   │
    ブランチで修正         PR作成可能
            │                   │
            └─────────┬─────────┘
                      │ マージ実行
                      ▼
┌─────────────────────────────────────────────────────────┐
│  main ブランチへマージ                                    │
└─────────────────────┬───────────────────────────────────┘
                      │ 自動トリガー
                      ▼
┌─────────────────────────────────────────────────────────┐
│  GitHub Actions: Deploy CI 実行                          │
│  - Quartoレンダリング                                     │
│  - GitHub Pagesデプロイ                                  │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│  GitHub Pages 公開完了                                   │
│  https://<username>.github.io/<repo>/                   │
└─────────────────────────────────────────────────────────┘
```

### ワークフロー1: Validation CI

**ファイル**: `.github/workflows/validate-content.yml`

**トリガー条件**:
```yaml
on:
  push:
    branches:
      - 'update/**'
      - 'fix/**'
      - 'config/**'
    paths:
      - 'raws/**/*.md'
      - 'raws/**/*.qmd'
      - '_quarto.yml'
  pull_request:
    branches: [main]
    paths:
      - 'raws/**/*.md'
      - 'raws/**/*.qmd'
      - '_quarto.yml'
```

**処理ステップ**:
1. リポジトリチェックアウト
2. Quartoセットアップ
3. **Quartoレンダリング検証** - 失敗時はワークフロー停止
4. リンク切れチェック（オプション、警告のみ）
5. Markdownリント（オプション、警告のみ）
6. プレビュー用HTMLをアーティファクトとして保存
7. 検証結果サマリー出力

**成功条件**:
- Quartoレンダリングが正常終了（exit code 0）

**失敗時の動作**:
- ❌ ワークフロー失敗
- ❌ PRマージ不可（Branch Protection Rulesにより）
- 📧 開発者に通知（GitHub標準機能）

### ワークフロー2: Deploy CI

**ファイル**: `.github/workflows/deploy-pages.yml`

**トリガー条件**:
```yaml
on:
  push:
    branches: [main]
    paths:
      - 'raws/**/*.md'
      - 'raws/**/*.qmd'
      - '_quarto.yml'
      - '.github/workflows/deploy-pages.yml'
  workflow_dispatch:  # 手動実行も可能
```

**処理ステップ**:
1. リポジトリチェックアウト
2. Quartoセットアップ
3. Quartoレンダリング実行
4. GitHub Pages設定
5. アーティファクトアップロード（`_site/`ディレクトリ）
6. GitHub Pagesデプロイ
7. デプロイ結果サマリー出力

**権限要件**:
```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

**並行実行制御**:
```yaml
concurrency:
  group: "pages"
  cancel-in-progress: false
```

---

## GitHub Branch Protection Rules

### 設定方法

GitHub WebUI: **Settings → Branches → Add branch protection rule**

### mainブランチ保護設定

```yaml
Branch name pattern: main

必須設定:
☑ Require a pull request before merging
  ☑ Require approvals: 0
    # 個人ブログの場合は0で可
    # チーム開発の場合は1以上推奨
  ☐ Dismiss stale pull request approvals
  ☐ Require review from Code Owners

☑ Require status checks to pass before merging
  ☑ Require branches to be up to date before merging
  Required status checks:
    - validate (Validation CIのジョブ名)

☑ Require conversation resolution before merging
  # PRコメントの解決を必須にする（オプション）

☑ Do not allow bypassing the above settings
  # 管理者でもルールをバイパスできないようにする

Allow force pushes:
  ☐ Everyone
  # Force pushを禁止

Allow deletions:
  ☐ Allow deletions
  # ブランチ削除を禁止
```

### 効果

- ❌ **mainへの直接push不可**
- ✅ **PR必須**
- ✅ **Validation CI通過必須**
- ✅ **mainブランチの健全性保証**

---

## 運用フロー

### 記事追加の標準フロー

```bash
# 1. mainブランチを最新に更新
git checkout main
git pull origin main

# 2. 作業ブランチ作成
git checkout -b update/add-quarto-intro

# 3. 記事を作成
cat > raws/quarto-intro.md << 'EOF'
---
title: "Quarto入門"
author: "Hawkie"
date: "2026-01-03"
categories: [tutorial, quarto]
---

# はじめに

Quartoの基本的な使い方を紹介します...
EOF

# 4. コミット & プッシュ
git add raws/quarto-intro.md
git commit -m "Add: Quarto入門記事を追加"
git push origin update/add-quarto-intro

# 5. GitHub で自動的に Validation CI 実行
#    ブラウザでGitHub Actionsタブを確認

# 6. Validation PASS後、GitHub WebUI でプルリクエスト作成
#    タイトル: "Add: Quarto入門記事"
#    本文: 記事の概要や変更内容

# 7. レビュー（オプション、個人ブログならスキップ可）

# 8. "Merge pull request" ボタンでマージ
#    → Deploy CI 自動実行
#    → 数分後にGitHub Pagesで公開

# 9. ローカルブランチ削除
git checkout main
git pull origin main
git branch -d update/add-quarto-intro
```

### エラー発生時の対処フロー

```bash
# Validation CI が失敗した場合

# 1. GitHub Actions のログでエラー内容確認
#    例: "YAML syntax error in raws/quarto-intro.md"

# 2. ローカルで修正
vim raws/quarto-intro.md
# YAMLフロントマターを修正

# 3. 再コミット & プッシュ
git add raws/quarto-intro.md
git commit -m "Fix: YAML syntax error"
git push origin update/add-quarto-intro

# 4. Validation CI 自動再実行
#    ✅ PASS → PRマージ可能

# mainブランチは一切汚染されていない
```

### 緊急修正フロー

```bash
# 公開済み記事に重大な誤りを発見した場合

# 1. 修正ブランチ作成
git checkout main
git pull origin main
git checkout -b fix/critical-error-in-post1

# 2. 修正
vim raws/post1.md

# 3. コミット & プッシュ
git add raws/post1.md
git commit -m "Fix: 重大な誤りを修正"
git push origin fix/critical-error-in-post1

# 4. Validation CI 通過後、即座にマージ
#    → Deploy CI 実行 → 公開

# 個人ブログの場合、Branch Protection Rulesで
# "Allow specified actors to bypass" に自分を追加しておけば
# 緊急時は直接mainにpush可能（非推奨）
```

---

## セキュリティ考慮事項

### Public Repository固有のリスク

本リポジトリは**Public**であるため、以下のリスクに注意が必要：

1. **すべてのコンテンツが公開される**
   - コミット履歴含め全世界に公開
   - 削除したファイルも履歴から復元可能

2. **GitHub Actionsログも公開**
   - ワークフローの実行ログが閲覧可能
   - 環境変数の誤出力に注意

3. **Fork可能**
   - 誰でもリポジトリをfork可能
   - 公開したくない情報は絶対に含めない

### 対策

#### 1. 機密情報の除外

**絶対に含めてはいけないもの**:
- ❌ APIキー、アクセストークン
- ❌ パスワード、シークレット
- ❌ 個人情報（住所、電話番号等）
- ❌ 非公開の業務情報
- ❌ `.env` ファイル

**`.gitignore` 設定**:
```gitignore
# Quarto
_site/
/.quarto/
_freeze/

# Secrets
.env
.env.local
*.secret
credentials.json

# System
.DS_Store
Thumbs.db
*.log
```

#### 2. GitHub Secrets使用時の注意

```yaml
# ❌ 悪い例: Secretを誤ってログ出力
- name: Bad Example
  run: echo "Token is ${{ secrets.API_TOKEN }}"
  # → ログにマスクされるが危険

# ✅ 良い例: Secretを環境変数として安全に使用
- name: Good Example
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: |
    # コマンド内でのみ使用
    curl -H "Authorization: Bearer $API_TOKEN" ...
```

#### 3. コンテンツレビュー

- ✅ PRレビュープロセスで公開前チェック
- ✅ 機密情報が含まれていないか確認
- ✅ 誤字脱字、リンク切れチェック

#### 4. 依存関係の管理

```yaml
# Quartoバージョン固定（セキュリティアップデート考慮）
- name: Setup Quarto
  uses: quarto-dev/quarto-actions/setup@v2
  with:
    version: 'release'  # または '1.4.x' で固定
```

#### 5. Actions権限の最小化

```yaml
permissions:
  contents: read      # 読み取りのみ
  pages: write        # Pages書き込み（必須）
  id-token: write     # OIDC認証（必須）
  # その他の権限は付与しない
```

---

## オプション機能

### 1. 自動PR作成

update/* ブランチへのpush時に自動的にPRを作成：

```yaml
# .github/workflows/auto-pr.yml
name: Auto Create PR

on:
  push:
    branches:
      - 'update/**'
      - 'fix/**'

jobs:
  create-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: Auto PR from ${{ github.ref_name }}
          title: "[${{ github.ref_name }}] Auto PR"
          body: |
            ## Auto-generated Pull Request

            Branch: `${{ github.ref_name }}`
            Commit: ${{ github.sha }}

            Please review and merge if validation passes.
          branch: ${{ github.ref_name }}
          base: main
```

### 2. リンク切れチェック

```yaml
# Validation CI に追加
- name: Check for Broken Links
  run: |
    npm install -g markdown-link-check
    find raws -name "*.md" -exec markdown-link-check {} \;
```

### 3. Markdownリント

```yaml
# Validation CI に追加
- name: Lint Markdown
  run: |
    npm install -g markdownlint-cli
    markdownlint 'raws/**/*.md'
```

### 4. コメント自動投稿

Validation結果をPRにコメント：

```yaml
- name: Comment PR
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: '✅ Validation passed! Ready to merge.'
      })
```

### 5. スケジュール実行

定期的な全記事再検証：

```yaml
on:
  schedule:
    - cron: '0 0 * * 0'  # 毎週日曜日00:00 (UTC)
```

### 6. Slack/Discord通知

```yaml
- name: Notify Slack
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
    payload: |
      {
        "text": "❌ Validation failed for ${{ github.ref_name }}"
      }
```

---

## Quarto設定詳細

### _quarto.yml

```yaml
project:
  type: website
  output-dir: _site

website:
  title: "Hawkie's Blog"
  description: "Technical memos and articles"

  navbar:
    background: primary
    left:
      - text: "Home"
        href: index.html
      - text: "Articles"
        href: articles.html
      - text: "About"
        href: about.html

  sidebar:
    style: "docked"
    search: true
    contents:
      - section: "Recent Posts"
        contents:
          - raws/*.md

execute:
  freeze: auto  # コード実行結果をキャッシュ

format:
  html:
    theme: cosmo
    css: styles.css
    toc: true
    toc-depth: 3
    toc-location: right
    code-copy: true
    code-fold: false
    code-tools: true
    link-external-newwindow: true
```

### カスタマイズ可能な項目

- **テーマ**: cosmo, flatly, minty, lumen, sandstone, etc.
- **構文ハイライト**: github, monokai, zenburn, etc.
- **フォント**: Google Fonts使用可能
- **レイアウト**: サイドバー、ナビゲーション構成

---

## トラブルシューティング

### Quartoレンダリング失敗

**症状**: Validation CIで `quarto render` が失敗

**原因と対策**:

1. **YAMLフロントマター構文エラー**
   ```yaml
   # ❌ 悪い例
   ---
   title: Article without quotes
   date: 2026-01-03  # 引用符なし
   ---

   # ✅ 良い例
   ---
   title: "Article Title"
   date: "2026-01-03"
   ---
   ```

2. **不正なMarkdown構文**
   - 閉じタグ忘れ（コードブロック、リンク等）
   - 特殊文字のエスケープ忘れ

3. **存在しないファイル参照**
   ```markdown
   # ❌ 存在しない画像
   ![image](./images/nonexistent.png)

   # ✅ 正しいパス
   ![image](./images/existing.png)
   ```

**デバッグ方法**:
```bash
# ローカルでQuartoレンダリング実行
quarto render raws/ --to html --verbose
```

### GitHub Pages公開されない

**症状**: Deploy CI成功するが、サイトが更新されない

**確認事項**:
1. GitHub Pages設定: Settings → Pages → Source = "GitHub Actions"
2. `_site/` ディレクトリが正しく生成されているか
3. ブラウザキャッシュクリア

### ブランチ保護ルール未適用

**症状**: mainへ直接pushできてしまう

**対策**:
1. Settings → Branches → Branch protection rules 再確認
2. "Do not allow bypassing" が有効か確認
3. 自分が "Allow specified actors" に含まれていないか確認

---

## 参考資料

- [Quarto公式ドキュメント](https://quarto.org/)
- [GitHub Actions公式ドキュメント](https://docs.github.com/actions)
- [GitHub Pages公式ドキュメント](https://docs.github.com/pages)
- [Branch Protection Rules](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)

---

## 変更履歴

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-03 | 初版作成 |

---

**End of Specification**
