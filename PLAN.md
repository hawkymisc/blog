# Quarto Markdown Publishing CI/CD Implementation Plan

**Version**: 1.0
**Last Updated**: 2026-01-03
**Target Branch**: `claude/setup-markdown-publishing-cicd-fHcfZ`

---

## 目次

1. [実装概要](#実装概要)
2. [実装フェーズ](#実装フェーズ)
3. [詳細タスクリスト](#詳細タスクリスト)
4. [依存関係](#依存関係)
5. [検証計画](#検証計画)
6. [ロールバック計画](#ロールバック計画)
7. [想定される問題と対策](#想定される問題と対策)

---

## 実装概要

### 目標

SPEC.mdで定義された仕様に基づき、Quarto Markdown Publishing CI/CDシステムを構築する。

### 実装スコープ

**含まれるもの**:
- ✅ Quarto設定ファイル作成
- ✅ Validation CI ワークフロー作成
- ✅ Deploy CI ワークフロー作成
- ✅ ディレクトリ構造整備
- ✅ サンプルコンテンツ作成
- ✅ `.gitignore` 更新
- ✅ ドキュメント整備（README更新）

**含まれないもの（将来対応）**:
- ❌ Branch Protection Rules設定（GitHub WebUIで手動設定）
- ❌ 自動PR作成機能
- ❌ リンク切れチェック機能
- ❌ Slack/Discord通知機能

### 実装方針

1. **段階的実装**: 小さく始めて、動作確認しながら進める
2. **最小構成**: 最初は必須機能のみ実装
3. **テスト駆動**: 各フェーズで動作検証を実施
4. **ドキュメント重視**: 実装と並行してドキュメント更新

---

## 実装フェーズ

### Phase 0: 準備 ✅

**目標**: 実装に必要な仕様・計画を整備

**タスク**:
- [x] SPEC.md 作成
- [x] PLAN.md 作成（本ドキュメント）

**成果物**:
- `SPEC.md`
- `PLAN.md`

---

### Phase 1: 基本構造整備

**目標**: Quartoプロジェクトの基本構造を構築

**タスク**:
1. `.gitignore` 更新
2. `raws/` ディレクトリ作成
3. `_quarto.yml` 設定ファイル作成
4. サンプルコンテンツ作成（`raws/index.md`）

**成果物**:
```
blog/
├── .gitignore (更新)
├── raws/
│   └── index.md
└── _quarto.yml
```

**検証方法**:
```bash
# ローカルでQuartoレンダリング実行
quarto render raws/ --to html

# _site/ ディレクトリが生成され、index.html が存在することを確認
ls -la _site/
```

**完了条件**:
- ✅ ローカルで `quarto render` が成功
- ✅ `_site/index.html` が正常に生成される

---

### Phase 2: Validation CI 実装

**目標**: update/* ブランチへのpush時に自動検証

**タスク**:
1. `.github/workflows/validate-content.yml` 作成
2. 基本的なQuartoレンダリング検証実装
3. エラーハンドリング追加
4. サマリー出力実装

**成果物**:
- `.github/workflows/validate-content.yml`

**検証方法**:
```bash
# テストブランチ作成
git checkout -b update/test-validation

# 意図的にエラーを含むファイル作成
cat > raws/test-error.md << 'EOF'
---
title: "Test Error
# ← YAMLエラー（閉じ引用符なし）
---
Test content
EOF

git add raws/test-error.md
git commit -m "Test: Validation CI error handling"
git push origin update/test-validation

# GitHub Actions で Validation CI が失敗することを確認

# エラー修正
cat > raws/test-error.md << 'EOF'
---
title: "Test Error"
---
Test content
EOF

git add raws/test-error.md
git commit -m "Fix: YAML syntax"
git push origin update/test-validation

# GitHub Actions で Validation CI が成功することを確認
```

**完了条件**:
- ✅ Validation CI が正常実行される
- ✅ エラー時に適切に失敗する
- ✅ 成功時にアーティファクトが生成される

---

### Phase 3: Deploy CI 実装

**目標**: mainブランチへのマージ時に自動デプロイ

**タスク**:
1. 既存 `.github/workflows/static.yml` を削除または無効化
2. `.github/workflows/deploy-pages.yml` 作成
3. GitHub Pages デプロイ設定
4. 権限設定（permissions）確認
5. 並行実行制御設定

**成果物**:
- `.github/workflows/deploy-pages.yml`
- `.github/workflows/static.yml` 削除

**検証方法**:
```bash
# update/test-validation ブランチをmainにマージ
# （この時点ではBranch Protection未設定なので直接可能）

git checkout main
git merge update/test-validation
git push origin main

# GitHub Actions で Deploy CI が実行されることを確認

# GitHub Pages で公開されることを確認
# https://<username>.github.io/<repo>/

# ブラウザで確認
```

**完了条件**:
- ✅ Deploy CI が正常実行される
- ✅ GitHub Pages が正常に更新される
- ✅ 公開URLでコンテンツが閲覧可能

---

### Phase 4: ドキュメント整備

**目標**: 運用に必要なドキュメントを整備

**タスク**:
1. README.md 更新
   - プロジェクト概要
   - セットアップ手順
   - 記事投稿手順
   - トラブルシューティング
2. CONTRIBUTING.md 作成（オプション）
3. サンプル記事追加

**成果物**:
- `README.md` (更新)
- `raws/` 配下にサンプル記事追加

**完了条件**:
- ✅ README.md に必要な情報が記載されている
- ✅ 新規ユーザーが記事を投稿できる

---

### Phase 5: Branch Protection Rules 設定 (手動)

**目標**: mainブランチの保護設定

**タスク**:
1. GitHub WebUI で Branch Protection Rules 設定
2. Validation CI を必須ステータスチェックに設定
3. 動作確認

**手順**:
1. GitHub リポジトリ → Settings → Branches
2. "Add branch protection rule" クリック
3. Branch name pattern: `main`
4. 以下を有効化:
   - ☑ Require a pull request before merging
   - ☑ Require status checks to pass before merging
     - Required checks: `validate`
   - ☑ Do not allow bypassing the above settings

**検証方法**:
```bash
# mainブランチへの直接pushが失敗することを確認
git checkout main
echo "test" >> README.md
git add README.md
git commit -m "Test: Direct push to main"
git push origin main
# → エラーが出ることを確認

# ブランチ経由のみ可能であることを確認
git checkout -b update/test-protection
git push origin update/test-protection
# → 成功することを確認
```

**完了条件**:
- ✅ mainへの直接push不可
- ✅ PR経由のみマージ可能
- ✅ Validation CI通過が必須

---

### Phase 6: 最終検証

**目標**: エンドツーエンドの動作確認

**タスク**:
1. 新規記事追加フローのテスト
2. エラーハンドリングのテスト
3. デプロイ確認
4. ドキュメント最終チェック

**検証シナリオ**:

#### シナリオ1: 正常系
```bash
git checkout -b update/add-first-article
cat > raws/first-article.md << 'EOF'
---
title: "初めての記事"
author: "Hawkie"
date: "2026-01-03"
---

# はじめに

これは初めての記事です。
EOF

git add raws/first-article.md
git commit -m "Add: 初めての記事"
git push origin update/add-first-article

# → Validation CI 成功確認
# → PR作成
# → マージ
# → Deploy CI 実行確認
# → GitHub Pages で公開確認
```

#### シナリオ2: 異常系
```bash
git checkout -b update/broken-article
cat > raws/broken.md << 'EOF'
---
title: Broken Article
date: invalid-date
---
EOF

git add raws/broken.md
git commit -m "Add: Broken article"
git push origin update/broken-article

# → Validation CI 失敗確認
# → PR作成不可確認
# → mainブランチが汚染されていないことを確認
```

**完了条件**:
- ✅ 全シナリオが想定通り動作
- ✅ エラーハンドリングが適切
- ✅ ドキュメントが正確

---

## 詳細タスクリスト

### Phase 1 タスク詳細

#### Task 1.1: .gitignore 更新

**ファイル**: `.gitignore`

**追加内容**:
```gitignore
# Quarto
_site/
/.quarto/
_freeze/

# Environment
.env
.env.local
*.secret

# System
.DS_Store
Thumbs.db
*.log
```

**検証**:
```bash
# Quartoレンダリング実行
quarto render raws/ --to html

# _site/ が git status に出ないことを確認
git status
```

---

#### Task 1.2: raws/ ディレクトリ作成

**コマンド**:
```bash
mkdir -p raws
```

---

#### Task 1.3: _quarto.yml 作成

**ファイル**: `_quarto.yml`

**内容**:
```yaml
project:
  type: website
  output-dir: _site

website:
  title: "Hawkie's Blog"
  description: "Personal memo and technical articles"

  navbar:
    background: primary
    left:
      - text: "Home"
        href: index.html
      - text: "About"
        href: about.html

execute:
  freeze: auto

format:
  html:
    theme: cosmo
    css: styles.css
    toc: true
    toc-depth: 3
    code-copy: true
    code-fold: false
    link-external-newwindow: true
```

---

#### Task 1.4: サンプルコンテンツ作成

**ファイル**: `raws/index.md`

**内容**:
```markdown
---
title: "Welcome to Hawkie's Blog"
---

# Home

This is my personal blog for technical memos and articles.

## About This Site

This site is built with:

- [Quarto](https://quarto.org/) - Scientific and technical publishing system
- [GitHub Pages](https://pages.github.com/) - Hosting
- [GitHub Actions](https://github.com/features/actions) - CI/CD

## Recent Posts

New posts will appear here automatically.
```

**ファイル**: `raws/about.md`

**内容**:
```markdown
---
title: "About"
---

# About Me

Hawkie's personal technical blog.

## Purpose

This blog serves as:

- Personal memo for technical topics
- Learning journal
- Knowledge sharing

## Contact

- GitHub: [Repository](https://github.com/<username>/<repo>)
```

---

### Phase 2 タスク詳細

#### Task 2.1: Validation CI ワークフロー作成

**ファイル**: `.github/workflows/validate-content.yml`

**内容**: (SPEC.md記載のワークフローを実装)

```yaml
name: Validate Content

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

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Quarto
        uses: quarto-dev/quarto-actions/setup@v2
        with:
          version: 'release'

      - name: Validate Quarto Rendering
        id: quarto_validate
        run: |
          echo "::group::Quarto Render Validation"
          quarto render raws/ --to html
          RESULT=$?
          echo "::endgroup::"

          if [ $RESULT -ne 0 ]; then
            echo "❌ Quarto rendering failed"
            exit 1
          else
            echo "✅ Quarto rendering successful"
          fi

      - name: Upload Preview Artifact
        uses: actions/upload-artifact@v4
        with:
          name: preview-site-${{ github.sha }}
          path: _site/
          retention-days: 7

      - name: Validation Summary
        if: always()
        run: |
          echo "## 🎯 Validation Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          if [ ${{ steps.quarto_validate.outcome }} == "success" ]; then
            echo "- ✅ Quarto Rendering: PASSED" >> $GITHUB_STEP_SUMMARY
            echo "- 📦 Preview artifact uploaded" >> $GITHUB_STEP_SUMMARY
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "**Ready to merge to main branch!**" >> $GITHUB_STEP_SUMMARY
          else
            echo "- ❌ Quarto Rendering: FAILED" >> $GITHUB_STEP_SUMMARY
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "**Please fix errors before merging.**" >> $GITHUB_STEP_SUMMARY
          fi
```

---

### Phase 3 タスク詳細

#### Task 3.1: 既存ワークフロー削除

**コマンド**:
```bash
git rm .github/workflows/static.yml
git commit -m "Remove old static deployment workflow"
```

---

#### Task 3.2: Deploy CI ワークフロー作成

**ファイル**: `.github/workflows/deploy-pages.yml`

**内容**:
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
    paths:
      - 'raws/**/*.md'
      - 'raws/**/*.qmd'
      - '_quarto.yml'
      - '.github/workflows/deploy-pages.yml'
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Quarto
        uses: quarto-dev/quarto-actions/setup@v2
        with:
          version: 'release'

      - name: Render Quarto Project
        run: |
          echo "::group::Rendering Quarto Project"
          quarto render raws/ --to html
          echo "::endgroup::"

      - name: Setup GitHub Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '_site'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4

      - name: Deployment Summary
        run: |
          echo "## 🚀 Deployment Successful" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Site URL**: ${{ steps.deployment.outputs.page_url }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "Deployed from commit: \`${{ github.sha }}\`" >> $GITHUB_STEP_SUMMARY
```

---

### Phase 4 タスク詳細

#### Task 4.1: README.md 更新

**ファイル**: `README.md`

**内容**: (既存内容を拡張)
```markdown
# blog

Hawkie's web-log. Main purpose is personal memo.

## 🚀 Features

- Built with [Quarto](https://quarto.org/)
- Automated publishing with GitHub Actions
- Hosted on GitHub Pages

## 📝 Adding New Posts

### 1. Create a new branch

```bash
git checkout -b update/your-article-name
```

### 2. Write your article

Create a new Markdown file in `raws/`:

```bash
cat > raws/your-article.md << 'EOF'
---
title: "Your Article Title"
author: "Your Name"
date: "2026-01-03"
categories: [category1, category2]
---

# Introduction

Your content here...
EOF
```

### 3. Commit and push

```bash
git add raws/your-article.md
git commit -m "Add: Your article title"
git push origin update/your-article-name
```

### 4. Create Pull Request

- Validation CI will automatically run
- If it passes, create a PR to `main`
- Merge the PR to publish

## 🏗️ Local Development

### Prerequisites

- [Quarto](https://quarto.org/docs/get-started/) installed

### Render locally

```bash
quarto render raws/ --to html
```

Open `_site/index.html` in your browser.

### Preview server

```bash
quarto preview raws/
```

## 📁 Directory Structure

```
blog/
├── raws/              # Markdown source files
├── _site/             # Generated HTML (gitignored)
├── _quarto.yml        # Quarto configuration
└── .github/workflows/ # CI/CD workflows
```

## 🔒 Security

This is a public repository. **Do not commit**:

- API keys or tokens
- Passwords or secrets
- Personal information
- Confidential data

## 📚 Documentation

- [SPEC.md](SPEC.md) - System specification
- [PLAN.md](PLAN.md) - Implementation plan

## 📄 License

See [LICENSE](LICENSE)
```

---

## 依存関係

### フェーズ間の依存関係

```
Phase 0 (準備)
    ↓
Phase 1 (基本構造) ← 必須
    ↓
Phase 2 (Validation CI) ← Phase 1完了後
    ↓
Phase 3 (Deploy CI) ← Phase 1完了後 (Phase 2と並行可能)
    ↓
Phase 4 (ドキュメント) ← Phase 1-3完了後
    ↓
Phase 5 (Branch Protection) ← Phase 2, 3完了後 (手動設定)
    ↓
Phase 6 (最終検証) ← Phase 1-5完了後
```

### 外部依存関係

- **Quarto**: バージョン 1.4以上推奨
- **GitHub Actions**:
  - `actions/checkout@v4`
  - `quarto-dev/quarto-actions/setup@v2`
  - `actions/configure-pages@v5`
  - `actions/upload-pages-artifact@v3`
  - `actions/deploy-pages@v4`
  - `actions/upload-artifact@v4`

### システム要件

- **GitHub Pages**: 有効化済み
- **GitHub Actions**: 有効化済み
- **リポジトリ権限**: Admin権限（Branch Protection設定のため）

---

## 検証計画

### 検証レベル

#### Level 1: ユニットテスト（各ファイル単位）

- **Quarto設定**: `quarto check` コマンド
- **Markdown構文**: Markdownパーサー
- **YAML構文**: YAMLバリデーター

#### Level 2: 統合テスト（ワークフロー単位）

- **Validation CI**: テストブランチでの実行確認
- **Deploy CI**: mainブランチでの実行確認

#### Level 3: エンドツーエンドテスト（全体フロー）

- **記事投稿フロー**: ブランチ作成 → PR → マージ → デプロイ
- **エラーハンドリング**: 意図的なエラーでの挙動確認

### 検証チェックリスト

#### Phase 1 検証

- [ ] `.gitignore` が正しく機能する
- [ ] `raws/` ディレクトリが作成される
- [ ] `_quarto.yml` が valid
- [ ] `quarto render` がローカルで成功
- [ ] `_site/index.html` が生成される
- [ ] HTMLが正常に表示される

#### Phase 2 検証

- [ ] Validation CI がトリガーされる
- [ ] 正常なMarkdownで成功する
- [ ] エラーのあるMarkdownで失敗する
- [ ] アーティファクトが生成される
- [ ] サマリーが表示される

#### Phase 3 検証

- [ ] Deploy CI がmainへのpushでトリガーされる
- [ ] `_site/` が正しくアップロードされる
- [ ] GitHub Pages が更新される
- [ ] 公開URLでアクセス可能
- [ ] 既存ワークフロー削除確認

#### Phase 4 検証

- [ ] README.md が分かりやすい
- [ ] サンプル記事が正常に表示される
- [ ] ドキュメントのリンクが切れていない

#### Phase 5 検証

- [ ] mainへの直接push不可
- [ ] PR経由のみマージ可能
- [ ] Validation CI通過が必須
- [ ] ルールがバイパスされない

#### Phase 6 検証

- [ ] 全シナリオが成功
- [ ] エラーハンドリング確認
- [ ] パフォーマンス確認（ビルド時間）

---

## ロールバック計画

### ロールバック戦略

各フェーズで問題が発生した場合のロールバック手順を定義。

#### Phase 1 失敗時

**症状**: Quartoレンダリングがローカルで失敗

**対応**:
```bash
# 追加ファイルを削除
git rm _quarto.yml
git rm -r raws/

# .gitignore を元に戻す
git checkout HEAD -- .gitignore

git commit -m "Rollback: Phase 1"
git push origin claude/setup-markdown-publishing-cicd-fHcfZ
```

#### Phase 2 失敗時

**症状**: Validation CI が動作しない

**対応**:
```bash
# ワークフローファイルを削除
git rm .github/workflows/validate-content.yml

git commit -m "Rollback: Phase 2 - Remove Validation CI"
git push origin claude/setup-markdown-publishing-cicd-fHcfZ
```

#### Phase 3 失敗時

**症状**: Deploy CI でエラー発生

**対応**:
```bash
# 新ワークフロー削除、旧ワークフロー復活
git rm .github/workflows/deploy-pages.yml
git checkout <previous-commit> -- .github/workflows/static.yml

git commit -m "Rollback: Phase 3 - Restore old deployment"
git push origin claude/setup-markdown-publishing-cicd-fHcfZ
```

**緊急対応**:
```bash
# GitHub Pages設定を手動でSourceを変更
# Settings → Pages → Source: Deploy from a branch (gh-pages or main)
```

#### Phase 5 失敗時

**症状**: Branch Protection Rulesで開発が止まる

**対応**:
- GitHub WebUI で Branch Protection Rules を無効化
- Settings → Branches → ルールを削除

### 完全ロールバック

**全フェーズをロールバック**:
```bash
# 実装前のコミットに戻る
git checkout main
git reset --hard <実装前のコミットハッシュ>
git push -f origin main

# 作業ブランチ削除
git push origin --delete claude/setup-markdown-publishing-cicd-fHcfZ
```

---

## 想定される問題と対策

### 問題1: Quartoインストールエラー

**症状**:
```
Error: Quarto installation failed
```

**原因**:
- GitHub Actionsのランナーで依存関係不足
- ネットワークエラー

**対策**:
```yaml
# Quartoバージョンを明示的に指定
- name: Setup Quarto
  uses: quarto-dev/quarto-actions/setup@v2
  with:
    version: '1.4.550'  # 固定バージョン
```

---

### 問題2: GitHub Pages デプロイ失敗

**症状**:
```
Error: Failed to create deployment
```

**原因**:
- GitHub Pages設定が正しくない
- 権限不足

**対策**:
1. Settings → Pages → Source: "GitHub Actions" に設定
2. ワークフローのpermissions確認:
   ```yaml
   permissions:
     contents: read
     pages: write
     id-token: write
   ```

---

### 問題3: アーティファクトアップロード失敗

**症状**:
```
Error: No files were found with the provided path: _site
```

**原因**:
- `_site/` ディレクトリが生成されていない
- Quartoレンダリング失敗が見逃された

**対策**:
```yaml
- name: Verify _site directory
  run: |
    if [ ! -d "_site" ]; then
      echo "Error: _site directory not found"
      exit 1
    fi
    ls -la _site/
```

---

### 問題4: Branch Protection Rules適用されない

**症状**:
- mainへ直接pushできてしまう

**原因**:
- ルール設定漏れ
- 管理者権限でバイパスされている

**対策**:
1. "Do not allow bypassing" を必ず有効化
2. 設定後、別アカウントでテスト

---

### 問題5: Validation CI がトリガーされない

**症状**:
- update/* ブランチへのpushでCIが動かない

**原因**:
- ブランチ名が命名規則に合っていない
- pathsフィルタに該当しない

**対策**:
```bash
# 正しいブランチ名を使用
git checkout -b update/article-name  # ✅
# NOT: git checkout -b feature/article-name  # ❌

# 対象ファイルを変更
git add raws/article.md  # ✅
# NOT: git add docs/article.md  # ❌（トリガーされない）
```

---

### 問題6: Quartoレンダリング時間が長い

**症状**:
- CI実行に10分以上かかる

**原因**:
- コード実行に時間がかかる
- キャッシュが効いていない

**対策**:
```yaml
# _quarto.yml でキャッシュ有効化
execute:
  freeze: auto  # コード実行結果をキャッシュ

# GitHub Actions でキャッシュ使用
- name: Cache Quarto
  uses: actions/cache@v3
  with:
    path: |
      ~/.quarto
      _freeze/
    key: ${{ runner.os }}-quarto-${{ hashFiles('_quarto.yml') }}
```

---

## タイムライン（参考）

実装にかかる時間の目安（実際の作業時間）:

| Phase | 所要時間 |
|-------|---------|
| Phase 0 | 完了 |
| Phase 1 | 15分 |
| Phase 2 | 20分 |
| Phase 3 | 20分 |
| Phase 4 | 15分 |
| Phase 5 | 5分（手動） |
| Phase 6 | 30分 |
| **合計** | **約2時間** |

※ トラブルシューティング時間は含まず

---

## 実装後の運用

### 定期メンテナンス

#### 週次
- [ ] GitHub Actionsのワークフロー実行状況確認
- [ ] エラーログのレビュー

#### 月次
- [ ] Quartoバージョン更新確認
- [ ] GitHub Actionsアクション更新確認
- [ ] リンク切れチェック

#### 四半期
- [ ] パフォーマンス分析
- [ ] セキュリティアップデート確認

### 監視項目

- **CI/CD成功率**: 95%以上維持
- **ビルド時間**: 5分以内
- **デプロイ時間**: 3分以内

---

## 完了基準

### Phase 1-6 全体完了条件

- ✅ すべてのフェーズの完了条件を満たす
- ✅ ドキュメントが整備されている
- ✅ エンドツーエンドテストが成功
- ✅ README.mdに運用手順が記載されている
- ✅ SPEC.md、PLAN.mdが最新状態

### プロジェクト完了条件

- ✅ mainブランチにマージ済み
- ✅ GitHub Pages で公開されている
- ✅ Branch Protection Rules 設定済み
- ✅ 最低1つの実記事が公開されている

---

## 次のステップ（将来対応）

実装完了後、以下の機能追加を検討:

1. **自動PR作成機能**
2. **リンク切れチェック**
3. **Markdownリント**
4. **Slack/Discord通知**
5. **コメント自動投稿**
6. **パフォーマンス最適化**
7. **SEO対策（sitemap.xml等）**
8. **RSS Feed生成**
9. **検索機能追加**
10. **アナリティクス統合**

---

**End of Implementation Plan**
