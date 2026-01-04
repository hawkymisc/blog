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

## 2026-01-04: develop ブランチと main ブランチの比較

### 実施内容

リモートブランチ `origin/develop` と `origin/main` を比較し、どちらが先行しているか確認。

### 比較結果

#### コミット数

```bash
$ git rev-list --count origin/develop
8

$ git rev-list --count origin/main
7
```

**develop が 1コミット多い**

#### 最新コミット日時

```bash
$ git log -1 --format="%ci %s" origin/develop
2026-01-03 17:21:09 +0900 Add: Create styles.css for Quarto customization

$ git log -1 --format="%ci %s" origin/main
2026-01-03 16:47:07 +0900 Fix: CodeRabbit configuration YAML parsing error
```

**develop が約34分新しい（17:21 vs 16:47）**

#### ファイル差分

```bash
$ git diff --stat origin/main origin/develop
 styles.css | 14 ++++++++++++++
 1 file changed, 14 insertions(+)
```

**develop には `styles.css` が追加されている**

#### ブランチの分岐点

```bash
$ git merge-base origin/develop origin/main
1745e061c0d41a6a613dc30674ec61e23cc13736
```

**共通の祖先**: コミット `1745e06` (activate Pages and publish from GitHub Actions)

#### ブランチ構造

```
1745e06 (共通の祖先)
   ├── develop: 6コミット → 3c05fc0 (2026-01-03 17:21:09)
   │    └── styles.css が追加されている
   │
   └── main:    5コミット → 10ff128 (2026-01-03 16:47:07)
```

両ブランチは同じ祖先から分岐し、似たようなコミットメッセージを持つが、**異なるコミットハッシュ**を持っている。これは別々の作業ブランチから作成されたことを示している。

### 結論

**`develop` ブランチが先行している**

理由：
1. ✅ コミット数が多い（8 vs 7）
2. ✅ 最新コミット日時が新しい（2026-01-03 17:21:09 vs 16:47:07）
3. ✅ `styles.css` という追加ファイルが存在

### 実施した対応

**`origin/develop` から新しいブランチを作成**

```bash
$ git checkout -b claude/check-remaining-work-from-develop-zIeEX origin/develop
branch 'claude/check-remaining-work-from-develop-zIeEX' set up to track 'origin/develop'.
Switched to a new branch 'claude/check-remaining-work-from-develop-zIeEX'
```

### 教訓

**新しいブランチを作成する際は、以下の手順で最も先行しているブランチを特定する：**

```bash
# 1. リモート情報を同期
git fetch origin

# 2. ブランチを比較
git log --oneline --graph --all --decorate

# 3. コミット数を確認
git rev-list --count origin/branch1
git rev-list --count origin/branch2

# 4. 最新コミット日時を確認
git log -1 --format="%ci %s" origin/branch1
git log -1 --format="%ci %s" origin/branch2

# 5. ファイル差分を確認
git diff --stat origin/branch1 origin/branch2

# 6. 共通の祖先を確認
git merge-base origin/branch1 origin/branch2

# 7. 各ブランチ固有のコミットを確認
git log --oneline $(git merge-base origin/branch1 origin/branch2)..origin/branch1
git log --oneline $(git merge-base origin/branch1 origin/branch2)..origin/branch2
```

### ベストプラクティス

```bash
# 最も先行しているブランチから新規ブランチを作成
git checkout -b new-feature-branch origin/most-advanced-branch

# 作業ブランチの命名規則（Claude Code）
# claude/<description>-<session-id>
# 例: claude/check-remaining-work-zIeEX
```

---

**Updated**: 2026-01-04
