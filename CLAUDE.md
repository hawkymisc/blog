# Claude Code 作業ログ

このファイルには、Claude Codeでの作業中に学んだことや注意点を記録します。

## 2026-01-04: git branch -a でリモートブランチが表示されない問題

### 問題

`git branch -a` を実行したとき、以下のブランチしか表示されませんでした：

```
* claude/check-remaining-work-zIeEX
  claude/setup-markdown-publishing-cicd-fHcfZ
  remotes/origin/claude/check-remaining-work-zIeEX
  remotes/origin/claude/setup-markdown-publishing-cicd-fHcfZ
```

実際には `main` と `develop` ブランチがリモートに存在していたにもかかわらず、表示されていませんでした。

### 原因

**`git fetch` を実行していなかったため、リモートブランチ情報がローカルに同期されていなかった。**

#### Git コマンドの挙動の違い

1. **`git branch -a`**
   - ローカルにキャッシュされているリモート追跡ブランチ情報を表示
   - リモートリポジトリには問い合わせない
   - `git fetch` を実行しないと最新情報は反映されない

2. **`git ls-remote --heads origin`**
   - リモートリポジトリに直接問い合わせる
   - 常に最新のブランチ情報を取得できる
   - ローカルのキャッシュには依存しない

3. **`git fetch origin`**
   - リモートの最新情報をローカルに同期
   - リモート追跡ブランチを更新
   - これを実行後、`git branch -a` で最新のリモートブランチが表示される

### 実際の動作

```bash
# 最初 - mainとdevelopが表示されない
$ git branch -a
* claude/check-remaining-work-zIeEX
  remotes/origin/claude/check-remaining-work-zIeEX
  remotes/origin/claude/setup-markdown-publishing-cicd-fHcfZ

# リモートに直接問い合わせ - mainとdevelopが見つかる
$ git ls-remote --heads origin
3c05fc001eae43925fdae11d56ff967e724f9eaf	refs/heads/develop
10ff1281d4f46f54d0f276a9e8524634fc5d1cc7	refs/heads/main

# fetchで同期
$ git fetch origin
From http://127.0.0.1:16110/git/hawkymisc/blog
 * [new branch]      develop    -> origin/develop
 * [new branch]      main       -> origin/main

# 再度確認 - mainとdevelopが表示される
$ git branch -a
* claude/check-remaining-work-zIeEX
  remotes/origin/claude/check-remaining-work-zIeEX
  remotes/origin/claude/setup-markdown-publishing-cicd-fHcfZ
  remotes/origin/develop
  remotes/origin/main
```

### 教訓

**リモートブランチを確認する際は、必ず最初に `git fetch` を実行する。**

特にClaude Codeのような環境では、セッション開始時にリポジトリの状態が古い可能性があるため：

```bash
# ベストプラクティス
git fetch origin           # まずリモート情報を同期
git branch -a              # すべてのブランチを確認
git status                 # 現在の状態を確認
```

### 参考コマンド

```bash
# リモートブランチ一覧（fetch不要、直接問い合わせ）
git ls-remote --heads origin

# リモート情報を同期してからブランチ一覧
git fetch origin && git branch -a

# 特定のリモートブランチをチェックアウト
git fetch origin
git checkout -b local-branch-name origin/remote-branch-name
```

---

**Updated**: 2026-01-04
