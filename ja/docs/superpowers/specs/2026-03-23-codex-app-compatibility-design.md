# Codex App 互換性: Worktree と finishing skill の適応

既存の Claude Code や Codex CLI の挙動を壊すことなく、Codex App の sandboxed worktree 環境でも superpowers skills が動作するようにします。

**Ticket:** PRI-823

## 動機

Codex App は、`$CODEX_HOME/worktrees/` 配下の detached HEAD な git worktree で agent を実行し、Seatbelt sandbox により `git checkout -b`、`git push`、network access をブロックします。3 つの superpowers skill は unrestricted な git access を前提にしています。`using-git-worktrees` は名前付き branch の手動 worktree を作成し、`finishing-a-development-branch` は branch 名を使って merge/push/PR を行い、`subagent-driven-development` はその両方を必要とします。

Codex CLI（オープンソースのターミナルツール）にはこの衝突は **ありません**。built-in の worktree management を持たないためです。そこで私たちの手動 worktree アプローチが isolation の不足を補っています。問題があるのは Codex App に限られます。

## 実測による知見

2026-03-23 に Codex App でテスト:

| Operation | workspace-write sandbox | Full access sandbox |
|---|---|---|
| `git add` | 動作する | 動作する |
| `git commit` | 動作する | 動作する |
| `git checkout -b` | **Blocked**（`.git/refs/heads/` に書き込めない） | 動作する |
| `git push` | **Blocked**（network + `.git/refs/remotes/`） | 動作する |
| `gh pr create` | **Blocked**（network） | 動作する |
| `git status/diff/log` | 動作する | 動作する |

追加の知見:
- `spawn_agent` の subagent は親 thread の filesystem を**共有**する（marker file test で確認）
- App header の "Create branch" button は、その worktree がどの branch から開始されたかに関係なく表示される
- App ネイティブの finishing flow: Create branch → Commit modal → Commit and push / Commit and create PR
- `network_access = true` config は macOS で黙って壊れている（issue #10390）

## 設計: 読み取り専用の環境検出

3 つの読み取り専用 git command により、副作用なしで環境を検出します。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

ここから 2 つのシグナルを導きます。

- **IN_LINKED_WORKTREE:** `GIT_DIR != GIT_COMMON` — 何者か（Codex App、Claude Code Agent tool、以前の skill 実行、またはユーザー）によって作られた worktree に agent がいる
- **ON_DETACHED_HEAD:** `BRANCH` が空 — 名前付き branch が存在しない

`show-toplevel` を見るのではなく `git-dir != git-common-dir` を使う理由:
- 通常の repo では、どちらも同じ `.git` directory を指す
- linked worktree では、`git-dir` は `.git/worktrees/<name>`、`git-common-dir` は `.git`
- submodule では両者は等しいため、`show-toplevel` で起きる false positive を避けられる
- `cd && pwd -P` で解決することで、relative-path 問題（通常 repo の `git-common-dir` は `.git` を相対で返すが、worktree では絶対パスになる）と symlink（macOS の `/tmp` → `/private/tmp`）の両方を扱える

### 意思決定マトリクス

| Linked Worktree? | Detached HEAD? | Environment | Action |
|---|---|---|---|
| No | No | Claude Code / Codex CLI / 通常の git | 完全な skill の挙動（変更なし） |
| Yes | Yes | Codex App worktree（workspace-write） | worktree 作成をスキップし、finish 時に handoff payload を出す |
| Yes | No | Codex App（Full access）または手動 worktree | worktree 作成をスキップし、完全な finishing flow を行う |
| No | Yes | 珍しいケース（手動の detached HEAD） | 通常どおり worktree を作成し、finish 時に警告する |

## 変更内容

### 1. `using-git-worktrees/SKILL.md` — Step 0 を追加（約 12 行）

"Overview" と "Directory Selection Process" の間に新セクションを追加:

**Step 0: すでに isolated workspace にいるか確認する**

検出 command を実行します。`GIT_DIR != GIT_COMMON` の場合、worktree 作成は完全にスキップします。代わりに:
1. Creation Steps 内の "Run Project Setup" 小節へ進む — `npm install` などは冪等であり、安全のため実行する価値がある
2. その後 "Verify Clean Baseline" — テストを実行する
3. branch の状態とともに報告する:
   - branch 上にいる場合: "Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD の場合: "Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

`GIT_DIR == GIT_COMMON` の場合は、完全な worktree 作成フローを継続します（変更なし）。

Step 0 が発火した場合、Safety verification（`.gitignore` チェック）はスキップされます。外部作成の worktree には無関係だからです。

Integration セクションの "Called by" 各項目も更新します。説明文を文脈依存の文言から、次へ変更します: "Ensures isolated workspace (creates one or verifies existing)"。たとえば `subagent-driven-development` の項目は、"REQUIRED: Set up isolated workspace before starting" から "REQUIRED: Ensures isolated workspace (creates one or verifies existing)" に変わります。

**Sandbox fallback:** `GIT_DIR == GIT_COMMON` で skill が Creation Steps に進んだものの、`git worktree add -b` が permission error（例: Seatbelt sandbox denial）で失敗した場合、これを「遅れて検出された制限付き環境」とみなします。Step 0 の「already in workspace」挙動へフォールバックし、作成をスキップし、現在の directory で setup と baseline tests を実行し、その旨を報告します。

Step 0 で報告したら、その時点で **STOP**。Directory Selection や Creation Steps へ進んではいけません。

**それ以外はすべて変更なし:** Directory Selection、Safety Verification、Creation Steps、Project Setup、Baseline Tests、Quick Reference、Common Mistakes、Red Flags。

### 2. `finishing-a-development-branch/SKILL.md` — Step 1.5 + cleanup guard を追加（約 20 行）

**Step 1.5: 環境を検出する**（Step 1 "Verify Tests" の後、Step 2 "Determine Base Branch" の前）

検出 command を実行します。3 つの経路があります。

- **Path A** は Step 2 と Step 3 を完全にスキップします（base branch も options も不要）。
- **Paths B と C** は通常どおり Step 2（Determine Base Branch）と Step 3（Present Options）へ進みます。

**Path A — 外部管理 worktree + detached HEAD**（`GIT_DIR != GIT_COMMON` かつ `BRANCH` が空）:

まず、すべての作業が stage・commit 済みであることを確認します（`git add` + `git commit`）。Codex App の finishing controls は commit 済みの作業を前提に動作します。

その後、ユーザーには次を提示します（4-option menu は**提示しない**こと）:

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

branch 名の導出: ticket ID があればそれを使い（例: `pri-823/codex-compat`）、なければ plan title の先頭 5 語を slugify し、それも無理なら提案自体を省略します。branch 名に機微な内容（脆弱性の説明、顧客名）を含めないようにします。

Step 5 へスキップします（外部管理 worktree では cleanup は no-op）。

**Path B — 外部管理 worktree + 名前付き branch**（`GIT_DIR != GIT_COMMON` かつ `BRANCH` が存在）:

通常どおり 4-option menu を提示します。（Step 5 の cleanup guard が外部管理状態を独立に再検出します。）

**Path C — 通常環境**（`GIT_DIR == GIT_COMMON`）:

現行どおりの 4-option menu を提示します（変更なし）。

**Step 5 cleanup guard:**

cleanup 時に `GIT_DIR` と `GIT_COMMON` の検出を再実行します（以前の skill 出力には依存しないこと。finishing skill は別セッションで実行される可能性があります）。`GIT_DIR != GIT_COMMON` なら `git worktree remove` はスキップします — この workspace の所有者は host environment です。

それ以外では、現行どおり確認して削除します。補足: 既存の Step 5 テキストは "For Options 1, 2, 4" と言っていますが、Quick Reference table と Common Mistakes section は "Options 1 & 4 only" と言っています。今回の新しい guard は既存ロジックの前に追加されるだけで、cleanup を発動させる option 自体は変えません。

**それ以外はすべて変更なし:** Options 1-4 のロジック、Quick Reference、Common Mistakes、Red Flags。

### 3. `subagent-driven-development/SKILL.md` と `executing-plans/SKILL.md` — 各 1 行編集

両方の skill に同一の Integration セクション行があります。次の変更を行います:
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
↓
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**それ以外はすべて変更なし:** Dispatch/review loop、prompt templates、model selection、status handling、red flags。

### 4. `codex-tools.md` — 環境検出ドキュメントを追加（約 15 行）

末尾に 2 つの新セクションを追加:

**Environment Detection:**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.
```

**Codex App Finishing:**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

## 変わらないこと

- `implementer-prompt.md`, `spec-reviewer-prompt.md`, `code-quality-reviewer-prompt.md` — subagent prompts は変更しない
- `executing-plans/SKILL.md` — 変更は Integration の 1 行説明のみ（`subagent-driven-development` と同じ）。実行時の挙動は一切変わらない
- `dispatching-parallel-agents/SKILL.md` — worktree や finishing 操作は行わない
- `.codex/INSTALL.md` — インストール手順は変更なし
- 4-option finishing menu — Claude Code と Codex CLI では完全に維持
- 完全な worktree 作成フロー — 非 worktree 環境では完全に維持
- subagent の dispatch/review/iterate loop — 変更なし（filesystem 共有は確認済み）

## スコープ要約

| File | Change |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + cleanup guard） |
| `skills/subagent-driven-development/SKILL.md` | 1 行編集 |
| `skills/executing-plans/SKILL.md` | 1 行編集 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 ファイル全体で約 50 行の追加/変更。新規ファイルはゼロ。破壊的変更もゼロ。

## 今後の検討事項

3 つ目の skill でも同じ検出パターンが必要になった場合は、共有の `references/environment-detection.md` ファイル（Approach B）へ切り出します。現時点では不要です — これを使う skill は 2 つだけです。

## テスト計画

### 自動テスト（実装後に Claude Code で実行）

1. 通常 repo の検出 — IN_LINKED_WORKTREE=false を確認
2. linked worktree の検出 — `git worktree add` で test worktree を作成し、IN_LINKED_WORKTREE=true を確認
3. detached HEAD の検出 — `git checkout --detach` を行い、ON_DETACHED_HEAD=true を確認
4. finishing skill の handoff 出力 — 制限付き環境で handoff message（4-option menu ではない）を確認
5. **Step 5 cleanup guard** — linked worktree を作成し（`git worktree add /tmp/test-cleanup -b test-cleanup`）、そこへ `cd` して Step 5 cleanup detection（`GIT_DIR` vs `GIT_COMMON`）を実行し、`git worktree remove` を **呼ばない** はずであることを確認する。その後 main repo に戻って同じ検出を実行し、今度は `git worktree remove` を **呼ぶ** はずであることを確認する。最後に test worktree を cleanup する。

### 手動 Codex App テスト（5 件）

1. Worktree thread（workspace-write）での検出 — GIT_DIR != GIT_COMMON と空 branch を確認
2. Worktree thread（Full access）での検出 — 同じ検出だが、sandbox 挙動が異なる
3. finishing skill の handoff format — agent が 4-option menu ではなく handoff payload を出すことを確認
4. フルライフサイクル — 検出 → commit → finishing detection → 正しい挙動 → cleanup
5. **Local thread での sandbox fallback** — Codex App の **Local thread**（workspace-write sandbox）を開始。Prompt: "Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change." 事前確認: `git checkout -b test-sandbox-check` は `Operation not permitted` で失敗するはず。期待結果: skill は `GIT_DIR == GIT_COMMON`（通常 repo）を検出し、`git worktree add -b` を試み、Seatbelt denial に当たり、Step 0 の "already in workspace" 挙動へフォールバックする — setup と baseline tests を実行し、現在の directory から ready を報告する。Pass: agent が cryptic error messages を出さずに自然に回復する。Fail: 生の Seatbelt error を出力する、再試行する、または分かりにくい出力で諦める。

### 回帰

- 既存の Claude Code の skill-triggering tests が引き続き通る
- 既存の subagent-driven-development integration tests が引き続き通る
- 通常の Claude Code セッションで、完全な worktree 作成 + 4-option finishing が引き続き動作する
