# Codex Tool 対応表

skills は Claude Code の tool 名を使っている。skill の中でこれらを見かけたら、あなたのプラットフォームでは次の対応物を使うこと:

| Skill references | Codex equivalent |
|-----------------|------------------|
| `Task` tool (dispatch subagent) | `spawn_agent`（[Subagent dispatch requires multi-agent support](#subagent-dispatch-requires-multi-agent-support) を参照） |
| Multiple `Task` calls (parallel) | 複数の `spawn_agent` 呼び出し |
| Task returns result | `wait_agent` |
| Task completes automatically | スロットを解放するために `close_agent` |
| `TodoWrite` (task tracking) | `update_plan` |
| `Skill` tool (invoke a skill) | skills はネイティブに読み込まれる — 指示にそのまま従う |
| `Read`, `Write`, `Edit` (files) | ネイティブの file tools を使う |
| `Bash` (run commands) | ネイティブの shell tools を使う |

## Subagent dispatch には multi-agent support が必要

Codex の設定（`~/.codex/config.toml`）に追加する:

```toml
[features]
multi_agent = true
```

これにより、`dispatching-parallel-agents` や `subagent-driven-development` などの skills で、`spawn_agent`、`wait_agent`、`close_agent` が有効になる。

Legacy note: `rust-v0.115.0` より前の Codex build では、spawn された agent を待つ機能は `wait` として公開されていた。現在の Codex では、spawn された agent に対して `wait_agent` を使う。`wait` という名前は現在、code-mode の `exec/wait` に使われており、`cell_id` で yield した exec cell を再開するためのもので、spawn された agent の結果取得用 tool ではない。

## 環境検出

worktree の作成や branch の仕上げを行う skills は、進める前に read-only の git command で環境を検出すべきである:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → すでに linked worktree 内にいる（作成をスキップする）
- `BRANCH` が空 → detached HEAD（sandbox から branch/push/PR はできない）

各 skill がこれらの signal をどう使うかは、`using-git-worktrees` の Step 0 と `finishing-a-development-branch` の Step 1 を参照。

## Codex App での仕上げ

sandbox により branch/push 操作がブロックされる場合（外部管理の worktree における detached HEAD）、agent はすべての作業を commit し、ユーザーに App のネイティブ controls を使うよう案内する:

- **"Create branch"** — branch 名を付け、その後 commit/push/PR は App UI で行う
- **"Hand off to local"** — 作業をユーザーのローカル checkout に引き渡す

agent は引き続き tests の実行、files の stage、branch 名・commit message・PR description の提案出力を行えるので、ユーザーはそれをコピーして使える。
