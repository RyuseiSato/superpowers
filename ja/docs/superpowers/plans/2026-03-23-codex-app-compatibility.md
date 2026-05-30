# Codex App Compatibility 実装計画

> **エージェントワーカー向け:** 必須サブスキル: この計画を task ごとに実装するには superpowers:subagent-driven-development（推奨）または superpowers:executing-plans を使用すること。追跡には checkbox (`- [ ]`) syntax を使う。

**目標:** `using-git-worktrees`、`finishing-a-development-branch`、および関連スキルが、既存の挙動を壊さずに Codex App の sandboxed worktree environment で動作するようにする。

**アーキテクチャ:** 2 つのスキルの冒頭に read-only な environment detection（`git-dir` vs `git-common-dir`）を追加する。すでに linked worktree にいる場合は作成をスキップする。detached HEAD 上では 4-option menu の代わりに handoff payload を出力する。sandbox fallback は worktree 作成時の permission error を捕捉する。

**技術スタック:** Git、Markdown（skill files は実行コードではなく instruction documents）

**仕様:** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## File Structure

| File | Responsibility | Action |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | Worktree creation + isolation | Step 0 detection + sandbox fallback を追加 |
| `skills/finishing-a-development-branch/SKILL.md` | Branch finishing workflow | Step 1.5 detection + cleanup guard を追加 |
| `skills/subagent-driven-development/SKILL.md` | Plan execution with subagents | Integration 説明を更新 |
| `skills/executing-plans/SKILL.md` | Plan execution inline | Integration 説明を更新 |
| `skills/using-superpowers/references/codex-tools.md` | Codex platform reference | detection + finishing docs を追加 |

---

### Task 1: `using-git-worktrees` に Step 0 を追加する

**Files:**
- Modify: `skills/using-git-worktrees/SKILL.md:14-15`（Overview の後、Directory Selection Process の前に挿入）

- [ ] **Step 1: 現在の skill file を読む**

`skills/using-git-worktrees/SKILL.md` 全体を読む。正確な挿入位置を特定する: "Announce at start" 行（14 行目）の後、"## Directory Selection Process"（16 行目）の前。

- [ ] **Step 2: Step 0 セクションを挿入する**

Overview セクションと "## Directory Selection Process" の間に、次を挿入する:

```markdown
## Step 0: Check if Already in an Isolated Workspace

Before creating a worktree, check if one already exists:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**If `GIT_DIR` differs from `GIT_COMMON`:** You are already inside a linked worktree (created by the Codex App, Claude Code's Agent tool, a previous skill run, or the user). Do NOT create another worktree. Instead:

1. Run project setup (auto-detect package manager as in "Run Project Setup" below)
2. Verify clean baseline (run tests as in "Verify Clean Baseline" below)
3. Report with branch state:
   - On a branch: "Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD: "Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

After reporting, STOP. Do not continue to Directory Selection or Creation Steps.

**If `GIT_DIR` equals `GIT_COMMON`:** Proceed with the full worktree creation flow below.

**Sandbox fallback:** If you proceed to Creation Steps but `git worktree add -b` fails with a permission error (e.g., "Operation not permitted"), treat this as a late-detected restricted environment. Fall back to the behavior above — run setup and baseline tests in the current directory, report accordingly, and STOP.
```

- [ ] **Step 3: 挿入内容を確認する**

ファイルを再度読み、次を確認する:
- Step 0 が Overview と Directory Selection Process の間にある
- ファイルの他の部分（Directory Selection、Safety Verification、Creation Steps など）は変更されていない
- 重複セクションや壊れた markdown がない

- [ ] **Step 4: Commit**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection (PRI-823)

Skip worktree creation when already in a linked worktree. Includes
sandbox fallback for permission errors on git worktree add."
```

---

### Task 2: `using-git-worktrees` の Integration セクションを更新する

**Files:**
- Modify: `skills/using-git-worktrees/SKILL.md:211-215`（Integration > Called by）

- [ ] **Step 1: 3 つの "Called by" エントリを更新する**

212-214 行を次から:

```markdown
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

次へ変更する:

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2: Integration セクションを確認する**

Integration セクションを読み、3 つのエントリすべてが更新され、"Pairs with" は変更されていないことを確認する。

- [ ] **Step 3: Commit**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions (PRI-823)

Clarify that skill ensures a workspace exists, not that it always creates one."
```

---

### Task 3: `finishing-a-development-branch` に Step 1.5 を追加する

**Files:**
- Modify: `skills/finishing-a-development-branch/SKILL.md:38`（Step 1 の後、Step 2 の前に挿入）

- [ ] **Step 1: 現在の skill file を読む**

`skills/finishing-a-development-branch/SKILL.md` 全体を読む。挿入位置を特定する: "**If tests pass:** Continue to Step 2."（38 行目）の後、"### Step 2: Determine Base Branch"（40 行目）の前。

- [ ] **Step 2: Step 1.5 セクションを挿入する**

Step 1 と Step 2 の間に、次を挿入する:

```markdown
### Step 1.5: Detect Environment

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Path A — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` is empty (externally managed worktree, detached HEAD):**

First, ensure all work is staged and committed (`git add` + `git commit`).

Then present this to the user (do NOT present the 4-option menu):

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

Branch name: use ticket ID if available (e.g., `pri-823/codex-compat`), otherwise slugify the first 5 words of the plan title, otherwise omit. Avoid sensitive content in branch names.

Skip to Step 5 (cleanup is a no-op — see guard below).

**Path B — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` exists (externally managed worktree, named branch):**

Proceed to Step 2 and present the 4-option menu as normal.

**Path C — `GIT_DIR` equals `GIT_COMMON` (normal environment):**

Proceed to Step 2 and present the 4-option menu as normal.
```

- [ ] **Step 3: 挿入内容を確認する**

ファイルを再度読み、次を確認する:
- Step 1.5 が Step 1 と Step 2 の間にある
- Steps 2-5 は変更されていない
- Path A の handoff に commit SHA とデータ消失警告が含まれている
- Paths B と C は通常どおり Step 2 に進む

- [ ] **Step 4: Commit**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection (PRI-823)

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu. Includes commit SHA and data loss warning."
```

---

### Task 4: `finishing-a-development-branch` に Step 5 cleanup guard を追加する

**Files:**
- Modify: `skills/finishing-a-development-branch/SKILL.md`（Step 5: Cleanup Worktree — Task 3 の後は行番号がずれるので section heading で探す）

- [ ] **Step 1: 現在の Step 5 セクションを読む**

`skills/finishing-a-development-branch/SKILL.md` の "### Step 5: Cleanup Worktree" セクションを探す（Task 3 の挿入後は行番号がずれている）。現在の Step 5 は次のとおり:

```markdown
### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

- [ ] **Step 2: 既存ロジックの前に cleanup guard を追加する**

Step 5 セクションを次の内容に置き換える:

```markdown
### Step 5: Cleanup Worktree

**First, check if worktree is externally managed:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

If `GIT_DIR` differs from `GIT_COMMON`: skip worktree removal — the host environment owns this workspace.

**Otherwise, for Options 1 and 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

Note: 元の文面は "For Options 1, 2, 4" だったが、Quick Reference table と Common Mistakes section は "Options 1 & 4 only" となっている。この edit は Step 5 をそれらのセクションに合わせるもの。

- [ ] **Step 3: 置き換え内容を確認する**

Step 5 を読み、次を確認する:
- cleanup guard（再検出）が先頭にある
- externally-managed でない worktree 向けの既存 removal logic が保持されている
- "Options 1 and 4"（"1, 2, 4" ではない）が Quick Reference と Common Mistakes に一致している

- [ ] **Step 4: Commit**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard (PRI-823)

Re-detect externally managed worktree at cleanup time and skip removal.
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only, matching Quick Reference and Common Mistakes."
```

---

### Task 5: `subagent-driven-development` と `executing-plans` の Integration 行を更新する

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md:268`
- Modify: `skills/executing-plans/SKILL.md:68`

- [ ] **Step 1: `subagent-driven-development` を更新する**

268 行を次から:
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
次へ:
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2: `executing-plans` を更新する**

68 行を次から:
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
次へ:
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 3: 両方の files を確認する**

`skills/subagent-driven-development/SKILL.md` の 268 行目と `skills/executing-plans/SKILL.md` の 68 行目を読み、両方とも "Ensures isolated workspace (creates one or verifies existing)" になっていることを確認する。

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): update worktree Integration descriptions (PRI-823)

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one."
```

---

### Task 6: `codex-tools.md` に environment detection docs を追加する

**Files:**
- Modify: `skills/using-superpowers/references/codex-tools.md:25`（末尾に追記）

- [ ] **Step 1: 現在の file を読む**

`skills/using-superpowers/references/codex-tools.md` 全体を読む。multi_agent section の後、25-26 行目あたりで終わっていることを確認する。

- [ ] **Step 2: 2 つの新しい section を末尾に追記する**

ファイル末尾に次を追加する:

```markdown

## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.

## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

- [ ] **Step 3: 追記内容を確認する**

ファイル全体を読み、次を確認する:
- 既存内容の後に 2 つの新しい section が現れる
- bash code block が正しく表示される（escape されていない）
- Step 0 と Step 1.5 への cross-reference がある

- [ ] **Step 4: Commit**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs (PRI-823)

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt."
```

---

### Task 7: Automated test — environment detection

**Files:**
- Create: `tests/codex-app-compat/test-environment-detection.sh`

- [ ] **Step 1: test directory を作成する**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **Step 2: detection test script を書く**

`tests/codex-app-compat/test-environment-detection.sh` を作成する:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Test environment detection logic from PRI-823
# Tests the git-dir vs git-common-dir comparison used by
# using-git-worktrees Step 0 and finishing-a-development-branch Step 1.5

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# Helper: run detection and return "linked" or "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== Test 1: Normal repo detection ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Normal repo detected as normal"
else
  log_fail "Normal repo detected as '$result' (expected 'normal')"
fi

echo "=== Test 2: Linked worktree detection ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree detected as linked"
else
  log_fail "Linked worktree detected as '$result' (expected 'linked')"
fi

echo "=== Test 3: Detached HEAD detection ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch is empty"
else
  log_fail "Detached HEAD: branch is '$branch' (expected empty)"
fi

echo "=== Test 4: Linked worktree + detached HEAD (Codex App simulation) ==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App simulation: linked + detached HEAD"
else
  log_fail "Codex App simulation: result='$result', branch='$branch'"
fi

echo "=== Test 5: Cleanup guard — linked worktree should NOT remove ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Cleanup guard: linked worktree correctly detected (would skip removal)"
else
  log_fail "Cleanup guard: expected 'linked', got '$result'"
fi

echo "=== Test 6: Cleanup guard — main repo SHOULD remove ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Cleanup guard: main repo correctly detected (would proceed with removal)"
else
  log_fail "Cleanup guard: expected 'normal', got '$result'"
fi

# Cleanup worktree before temp dir removal
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== Results: $PASS passed, $FAIL failed ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **Step 3: executable にして実行する**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

Expected output: 6 passed, 0 failed.

- [ ] **Step 4: Commit**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat (PRI-823)

Tests git-dir vs git-common-dir comparison in normal repo, linked
worktree, detached HEAD, and cleanup guard scenarios."
```

---

### Task 8: 最終確認

**Files:**
- Read: 変更した 5 つの skill files すべて

- [ ] **Step 1: automated detection tests を実行する**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

Expected: 6 passed, 0 failed.

- [ ] **Step 2: 変更した各 file を読み、内容を確認する**

各 file を end-to-end で読む:
- `skills/using-git-worktrees/SKILL.md` — Step 0 があり、残りは変更されていない
- `skills/finishing-a-development-branch/SKILL.md` — Step 1.5 があり、cleanup guard もあり、残りは変更されていない
- `skills/subagent-driven-development/SKILL.md` — 268 行目が更新されている
- `skills/executing-plans/SKILL.md` — 68 行目が更新されている
- `skills/using-superpowers/references/codex-tools.md` — 末尾に 2 つの新しい section がある

- [ ] **Step 3: 意図しない変更がないことを確認する**

```bash
git diff --stat HEAD~7
```

6 files changed（5 skill files + 1 test file）だけが表示されるはず。他の files は変更されていないこと。

- [ ] **Step 4: 既存 test suite を実行する**

もし test runner があれば:
```bash
# Run skill-triggering tests
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "Skill triggering tests not available in this environment"

# Run SDD integration test
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "SDD integration test not available in this environment"
```

Note: これらの tests は `--dangerously-skip-permissions` 付きの Claude Code を必要とする。利用できない場合は、regression tests を手動で実行すべきことを記録する。

