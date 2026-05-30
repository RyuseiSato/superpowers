# Copilot CLI Tool 対応表

skills は Claude Code の tool 名を使っている。skill の中でこれらを見かけたら、Copilot CLI では次の対応物を使うこと:

| Skill references | Copilot CLI equivalent |
|-----------------|----------------------|
| `Read` (file reading) | `view` |
| `Write` (file creation) | `create` |
| `Edit` (file editing) | `edit` |
| `Bash` (run commands) | `bash` |
| `Grep` (search file content) | `grep` |
| `Glob` (search files by name) | `glob` |
| `Skill` tool (invoke a skill) | `skill` |
| `WebFetch` | `web_fetch` |
| `Task` tool (dispatch subagent) | `task`（`agent_type: "general-purpose"` または `"explore"`） |
| Multiple `Task` calls (parallel) | 複数の `task` 呼び出し |
| Task status/output | `read_agent`, `list_agents` |
| `TodoWrite` (task tracking) | 組み込みの `todos` table を使う `sql` |
| `WebSearch` | 対応物なし — 検索エンジンの URL と一緒に `web_fetch` を使う |
| `EnterPlanMode` / `ExitPlanMode` | 対応物なし — メインセッションにとどまる |

## Async shell sessions

Copilot CLI は永続的な async shell session をサポートしており、これには Claude Code に直接対応するものがない:

| Tool | Purpose |
|------|---------|
| `bash` with `async: true` | 長時間実行コマンドをバックグラウンドで開始する |
| `write_bash` | 実行中の async session に入力を送る |
| `read_bash` | 実行中の async session から出力を読む |
| `stop_bash` | async session を終了する |
| `list_bash` | アクティブな shell session を一覧表示する |

## Copilot CLI の追加 tools

| Tool | Purpose |
|------|---------|
| `store_memory` | 将来のセッションのためにコードベースに関する事実を保存する |
| `report_intent` | 現在の intent で UI の status line を更新する |
| `sql` | セッションの SQLite database（todos、metadata）に問い合わせる |
| `fetch_copilot_cli_documentation` | Copilot CLI のドキュメントを参照する |
| GitHub MCP tools (`github-mcp-server-*`) | GitHub API へのネイティブアクセス（issues、PRs、code search） |
