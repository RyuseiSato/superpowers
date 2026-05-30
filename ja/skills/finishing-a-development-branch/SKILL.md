---
name: finishing-a-development-branch
description: 実装が完了し、すべてのテストが通り、その作業をどう統合するか決める必要があるときに使う。merge、PR、またはクリーンアップのための構造化された選択肢を提示して開発の完了を導く
---

# Finishing a Development Branch

## 概要

明確な選択肢を提示し、選ばれたワークフローを処理することで、開発作業の完了を導きます。

**中核原則:** テストを確認 → 環境を検出 → 選択肢を提示 → 選択を実行 → クリーンアップ。

**開始時に宣言する:** "I'm using the finishing-a-development-branch skill to complete this work."

## プロセス

### Step 1: テストを確認する

**選択肢を提示する前に、テストが通ることを確認してください:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**テストが失敗した場合:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

ここで停止します。Step 2 に進んではいけません。

**テストが通った場合:** Step 2 へ進みます。

### Step 2: 環境を検出する

**選択肢を提示する前に workspace の状態を判断します:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

これにより、どのメニューを表示し、クリーンアップをどう行うかが決まります:

| 状態 | メニュー | クリーンアップ |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（通常の repo） | 標準の 4 選択肢 | クリーンアップ対象の worktree なし |
| `GIT_DIR != GIT_COMMON`、名前付きブランチ | 標準の 4 選択肢 | provenance ベース（Step 6 参照） |
| `GIT_DIR != GIT_COMMON`、detached HEAD | 縮小版の 3 選択肢（merge なし） | クリーンアップなし（外部管理） |

### Step 3: ベースブランチを決める

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

または、こう尋ねます: "This branch split from main - is that correct?"

### Step 4: 選択肢を提示する

**通常 repo と名前付きブランチの worktree では、必ず次の 4 つをそのまま提示します:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD では、必ず次の 3 つだけを提示します:**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**説明を足さないこと** - 選択肢は簡潔に保ちます。

### Step 5: 選択を実行する

#### Option 1: ローカルで merge

```bash
# Get main repo root for CWD safety
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first — verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>

# Verify tests on merged result
<test command>

# Only after merge succeeds: cleanup worktree (Step 6), then delete branch
```

その後: worktree をクリーンアップ（Step 6）し、次にブランチを削除します:

```bash
git branch -d <feature-branch>
```

#### Option 2: Push して PR を作成する

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**worktree をクリーンアップしてはいけません** — ユーザーは PR フィードバックへの対応でそれを使い続ける必要があります。

#### Option 3: そのまま保持する

次のように報告します: "Keeping branch <name>. Worktree preserved at <path>."

**worktree をクリーンアップしないこと。**

#### Option 4: 破棄する

**まず確認する:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

正確な確認入力を待ちます。

確認されたら:
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

その後: worktree をクリーンアップ（Step 6）し、次にブランチを強制削除します:
```bash
git branch -D <feature-branch>
```

### Step 6: Workspace をクリーンアップする

**Option 1 と 4 のときだけ実行します。** Option 2 と 3 では常に worktree を保持します。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**`GIT_DIR == GIT_COMMON` の場合:** 通常 repo なので、クリーンアップすべき worktree はありません。完了です。

**worktree path が `.worktrees/`、`worktrees/`、または `~/.config/superpowers/worktrees/` 配下の場合:** Superpowers が作成した worktree なので、こちらがクリーンアップを担当します。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**それ以外の場合:** ホスト環境（ハーネス）がこの workspace を所有しています。削除してはいけません。プラットフォームに workspace-exit ツールがあるならそれを使います。なければそのまま残します。

## クイックリファレンス

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | yes | - | - | yes |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | yes (force) |

## よくあるミス

**テスト確認を飛ばす**
- **問題:** 壊れたコードを merge したり、失敗する PR を作ったりする
- **修正:** 選択肢を出す前に必ずテストを確認する

**自由回答の質問をする**
- **問題:** "What should I do next?" は曖昧
- **修正:** 必ず 4 つの構造化された選択肢（detached HEAD なら 3 つ）を提示する

**Option 2 で worktree をクリーンアップする**
- **問題:** PR の反復に必要な worktree を消してしまう
- **修正:** クリーンアップするのは Option 1 と 4 だけ

**worktree を消す前にブランチを削除する**
- **問題:** worktree がまだブランチを参照しているので `git branch -d` が失敗する
- **修正:** まず merge、次に worktree を削除、その後でブランチを削除する

**worktree の中から `git worktree remove` を実行する**
- **問題:** 削除対象 worktree の中に CWD があるとコマンドが黙って失敗する
- **修正:** `git worktree remove` 前に必ず main repo root へ `cd` する

**ハーネス所有の worktree をクリーンアップする**
- **問題:** ハーネスが作成した worktree を消すと phantom state を引き起こす
- **修正:** `.worktrees/`、`worktrees/`、`~/.config/superpowers/worktrees/` 配下の worktree だけをクリーンアップする

**破棄に確認を取らない**
- **問題:** 作業を誤って削除する
- **修正:** `discard` と入力させて確認を必須にする

## レッドフラグ

**絶対にしないこと:**
- テストが失敗しているのに進む
- merge 結果のテスト確認なしに merge する
- 確認なしに作業を削除する
- 明示的な依頼なしに force-push する
- merge 成功確認前に worktree を削除する
- 自分が作っていない worktree をクリーンアップする（provenance check）
- worktree の中から `git worktree remove` を実行する

**常に行うこと:**
- 選択肢を出す前にテストを確認する
- メニュー提示前に環境を検出する
- 必ずちょうど 4 つの選択肢（detached HEAD なら 3 つ）を提示する
- Option 4 では typed confirmation を得る
- worktree クリーンアップは Option 1 と 4 だけで行う
- worktree 削除前に main repo root へ `cd` する
- 削除後に `git worktree prune` を実行する

