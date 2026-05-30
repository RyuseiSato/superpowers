# Gemini CLI Tool 対応表

skills は Claude Code の tool 名を使っている。skill の中でこれらを見かけたら、Gemini CLI では次の対応物を使うこと:

| Skill references | Gemini CLI equivalent |
|-----------------|----------------------|
| `Read` (file reading) | `read_file` |
| `Write` (file creation) | `write_file` |
| `Edit` (file editing) | `replace` |
| `Bash` (run commands) | `run_shell_command` |
| `Grep` (search file content) | `grep_search` |
| `Glob` (search files by name) | `glob` |
| `TodoWrite` (task tracking) | `write_todos` |
| `Skill` tool (invoke a skill) | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` tool (dispatch subagent) | `@agent-name`（[Subagent support](#subagent-support) を参照） |

## Subagent support

Gemini CLI は `@` 構文による subagent をネイティブにサポートしている。あらゆるタスクの dispatch には、全 tools にアクセスでき、与えられた prompt に従う組み込み agent `@generalist` を使う。

skill が名前付き agent type の dispatch を指示している場合は、skill の prompt template にある完全な prompt とともに `@generalist` を使う:

| Skill instruction | Gemini CLI equivalent |
|-------------------|----------------------|
| `Task tool (superpowers:implementer)` | 値を埋めた `implementer-prompt.md` template とともに `@generalist` |
| `Task tool (superpowers:spec-reviewer)` | 値を埋めた `spec-reviewer-prompt.md` template とともに `@generalist` |
| `Task tool (superpowers:code-reviewer)` | `@code-reviewer`（同梱 agent）または、値を埋めた review prompt とともに `@generalist` |
| `Task tool (superpowers:code-quality-reviewer)` | 値を埋めた `code-quality-reviewer-prompt.md` template とともに `@generalist` |
| `Task tool (general-purpose)` with inline prompt | inline prompt とともに `@generalist` |

### Prompt の埋め込み

skills は `{WHAT_WAS_IMPLEMENTED}` や `[FULL TEXT of task]` のようなプレースホルダーを含む prompt template を提供する。すべてのプレースホルダーを埋め、完全な prompt を `@generalist` へのメッセージとして渡すこと。prompt template 自体に agent の役割、レビュー基準、期待される出力形式が含まれているため、`@generalist` はそれに従う。

### 並列 dispatch

Gemini CLI は並列の subagent dispatch をサポートしている。skill が複数の独立した subagent task を並列に dispatch するよう求める場合は、それらすべての `@generalist` または名前付き subagent task を、同じ prompt の中でまとめて依頼する。依存関係のある task は逐次実行にするが、履歴を単純に保つためだけに独立した subagent task を直列化してはならない。

## Gemini CLI の追加 tools

これらの tools は Gemini CLI で利用できるが、Claude Code には対応物がない:

| Tool | Purpose |
|------|---------|
| `list_directory` | files と subdirectories を一覧表示する |
| `save_memory` | セッションをまたいで事実を GEMINI.md に保存する |
| `ask_user` | ユーザーに構造化入力を求める |
| `tracker_create_task` | 高機能な task 管理（作成、更新、一覧、可視化） |
| `enter_plan_mode` / `exit_plan_mode` | 変更を加える前に read-only research mode へ切り替える |
