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
3c05fc001eae43925fdae11d56ff967e724f9eaf refs/heads/develop
10ff1281d4f46f54d0f276a9e8524634fc5d1cc7 refs/heads/main

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

#### ⚠️ 重要: コミットハッシュは時間とともに変わる

**上記のコミットハッシュ（`1745e06`、`3c05fc0`、`10ff128`など）は、このセッション時点のものです。**

ブランチが進化すると：
- 新しいコミットが追加される
- マージやリベースにより構造が変わる
- 共通の祖先も変化する可能性がある

**そのため、常に動的にブランチ状態を確認する必要があります。**

#### 分岐構造が変わった場合の対応

##### シナリオ1: mainとdevelopが進化している

```bash
# 現在の状態を確認
$ git fetch origin
$ git log --oneline --graph origin/develop origin/main -10

# 例: 両方とも進化している場合
* 7a8b9c0 (origin/develop) Add: New feature X
* 4d5e6f0 Update: Improve performance
| * 2b3c4d5 (origin/main) Fix: Bug in deployment
| * 9e8f7g6 Update: Security patch
|/
* 1a2b3c4 (共通の祖先)

# 共通の祖先を動的に確認
$ git merge-base origin/develop origin/main
1a2b3c4...  # ← この値は変わる可能性がある
```

##### シナリオ2: developがmainにマージされた後

```bash
# developの変更がmainにマージされると構造が変わる
$ git log --oneline --graph origin/develop origin/main -10

* 8h9i0j1 (origin/main) Merge branch 'develop'
|\
| * 3c05fc0 (origin/develop) Add: Create styles.css
|/
* 1745e06 activate Pages

# この場合、developとmainは再び同期される
$ git merge-base origin/develop origin/main
8h9i0j1...  # ← マージコミットが新しい共通祖先になる
```

##### シナリオ3: コミットハッシュが完全に異なる

```bash
# リベースやフォースプッシュがあった場合
$ git fetch origin
warning: fetch updated the current branch head.
fast-forwarding your working tree...

# ブランチ構造を再確認
$ git log --oneline --graph --all --decorate -20

# 必要に応じて、再度どちらのブランチが先行しているか判断
```

#### 対応方法の原則

**✅ 正しい対応:**
1. **常に `git merge-base` を使用** - ハードコードされたハッシュに依存しない
2. **動的にブランチ状態を確認** - セッション開始時に必ず `git fetch` と比較を実施
3. **最新コミット日時とコミット数で判断** - ハッシュではなく、これらの情報で先行度を判断

**❌ 避けるべき対応:**
1. ~~過去のコミットハッシュをハードコード~~
2. ~~一度の確認結果に依存~~
3. ~~視覚的なグラフだけで判断~~

#### 汎用的なブランチ比較スクリプト

```bash
#!/bin/bash
# ブランチ比較スクリプト（汎用版）

BRANCH1=${1:-origin/develop}
BRANCH2=${2:-origin/main}

echo "=== ブランチ比較: $BRANCH1 vs $BRANCH2 ==="
echo

# リモート情報を同期
git fetch origin

# コミット数
COUNT1=$(git rev-list --count $BRANCH1)
COUNT2=$(git rev-list --count $BRANCH2)
echo "コミット数:"
echo "  $BRANCH1: $COUNT1"
echo "  $BRANCH2: $COUNT2"
echo

# 最新コミット日時
LATEST1=$(git log -1 --format="%ci %s" $BRANCH1)
LATEST2=$(git log -1 --format="%ci %s" $BRANCH2)
echo "最新コミット:"
echo "  $BRANCH1: $LATEST1"
echo "  $BRANCH2: $LATEST2"
echo

# 共通の祖先
MERGE_BASE=$(git merge-base $BRANCH1 $BRANCH2)
echo "共通の祖先: $MERGE_BASE"
echo "  $(git log -1 --format='%s' $MERGE_BASE)"
echo

# 各ブランチ固有のコミット数
COMMITS_ONLY_1=$(git rev-list --count $MERGE_BASE..$BRANCH1)
COMMITS_ONLY_2=$(git rev-list --count $MERGE_BASE..$BRANCH2)
echo "分岐後のコミット数:"
echo "  $BRANCH1: $COMMITS_ONLY_1"
echo "  $BRANCH2: $COMMITS_ONLY_2"
echo

# ファイル差分
echo "ファイル差分:"
git diff --stat $BRANCH1 $BRANCH2
echo

# 結論
if [ $COUNT1 -gt $COUNT2 ]; then
    echo "結論: $BRANCH1 が先行しています"
elif [ $COUNT2 -gt $COUNT1 ]; then
    echo "結論: $BRANCH2 が先行しています"
else
    echo "結論: 同じコミット数です。日時で判断してください"
fi
```

使用方法:
```bash
# デフォルト（develop vs main）
$ ./compare-branches.sh

# 任意のブランチを比較
$ ./compare-branches.sh origin/feature origin/develop
```

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
