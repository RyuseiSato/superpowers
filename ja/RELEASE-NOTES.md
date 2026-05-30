# Superpowers リリースノート

## v5.1.0 (2026-04-30)

### 削除

- **従来の slash commands を削除** — `/brainstorm`、`/execute-plan`、`/write-plan` は廃止されました。これらは対応する skill を呼ぶようユーザーに伝えるだけの非推奨 stub でした。代わりに `superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans` を直接呼び出してください。(#1188)
- **`superpowers:code-reviewer` named agent を削除** — この agent は plugin 唯一の named agent で、利用していたのは 2 つの skills だけでした。一方、repo 内の他の reviewer/implementer subagent はすべて、skill と一緒に prompt template を渡して `general-purpose` を dispatch しています。agent の persona と checklist は、自己完結した Task-dispatch template として `skills/requesting-code-review/code-reviewer.md` に統合されました。`Task (superpowers:code-reviewer)` を dispatch していた場合は、代わりに prompt template 付きの `Task (general-purpose)` に切り替えてください。(PR #1299)
- **skills から integration sections を削除** — これらは agents に native skills system がなかった時代の名残であり、steering の助けにもなっていませんでした。

### Worktree Skills Rewrite

`using-git-worktrees` と `finishing-a-development-branch` は、agent がすでに分離された worktree 内で実行中かどうかを検出し、`git worktree` にフォールバックする前に harness の native worktree controls を優先するようになりました。振る舞いは TDD で検証され、5 つの harness で cross-platform 確認済みです。(PRI-974, PR #1121)

- **環境検出** — 両 skill は何かを行う前に `GIT_DIR != GIT_COMMON` を確認します。すでに linked worktree 内にいる場合、作成は完全にスキップされます。submodule guard により誤検出も防ぎます。
- **worktree 作成前の同意** — `using-git-worktrees` は暗黙的に worktree を作成しなくなりました。skill はまずユーザーに確認します。#991（subagent-driven-development が同意なく worktree を自動作成していた問題）を修正しました。
- **native tool の優先（Step 1a）** — harness が独自の worktree tool（例: Codex）を公開している場合、skill はそれを優先します。ユーザーが明示した希望も尊重されます。
- **由来ベースの cleanup** — `finishing-a-development-branch` は `.worktrees/`（superpowers が作成したもの）内の worktree のみを cleanup します。外部のものはそのまま残します。#940（Option 2 が誤って worktree を cleanup していた）、#999（merge-then-remove の順序）、#238（`git worktree remove` の前に repo root へ `cd`）を修正しました。
- **Detached HEAD の扱い** — merge 元の branch がない場合、finishing menu は 2 つの option に縮小されます。
- skill 例にあった **ハードコードされた `/Users/jesse` path** を汎用 placeholder に置き換えました。(#858, PR #1122)

### AI Agents 向け Contributor Guidelines

`CLAUDE.md`（`AGENTS.md` への symlink）の先頭に、AI agents に直接語りかける 2 つの新しい section が追加されました。この repo に対する直近 100 件の closed PR を監査したところ、AI 生成の slop が原因で 94% が reject されていました。PR template を読まない、duplicate を開く、problem description を捏造する、fork 固有または domain 固有の変更を upstream に push するといった行為が原因です。

- **提出前 checklist** — PR template を読む、既存 PR を検索する、実際の problem が存在することを確認する、変更が core に属することを確認する、提出前に完全な diff を human partner に見せる。
- **受け入れないもの** — third-party dependencies、skill 内容の「compliance」rewrite、project-specific configuration、大量 PR、推測ベースの修正、domain-specific skills、fork-specific changes、捏造された内容、無関係な変更の抱き合わせ。
- **新しい harness の PR には session transcript が必要** — 過去の新規 harness integration の多くは、session 開始時に `using-superpowers` bootstrap を読み込まず、skill files をコピーしたり `npx skills` でラップしただけでした。acceptance test（クリーンな session で "Let's make a react todo list" が `brainstorming` を自動 trigger すること）と完全な transcript が必須になりました。

### Codex Plugin Mirror Tooling

新しい `sync-to-codex-plugin` script により、superpowers を OpenAI Codex plugin marketplace の `prime-radiant-inc/openai-codex-plugins` へ mirror できるようになりました。path/user 非依存なので、どの team member でも実行できます。(PR #1165)

- 実行ごとに temp directory へ fork を fresh clone し、overlay を inline で再生成して PR を開きます。script 自身の場所から upstream を自動検出し、`rsync` / `git` / `gh auth` / `python3` を事前確認します。
- 初回セットアップ用の `--bootstrap` flag、source root に anchored された `EXCLUDES` pattern、`assets/` の除外。
- `CODE_OF_CONDUCT.md` を mirror し、`agents/openai.yaml` overlay を削除。
- mirror された `plugin.json` の `interface.defaultPrompt` を seed。(PR #1180 by @arittr)
- Codex plugin files は source repo に commit されているため、sync script は canonical version を使用します。Codex marketplace metadata も保持されます。

### OpenCode

- **bootstrap content を module level で cache** — `getBootstrapContent()` は OpenCode の agent loop で毎 step 発火する `experimental.chat.messages.transform` hook のたびに、`fs.existsSync` + `fs.readFileSync` + frontmatter regex を呼んでいました。これを session の寿命中 1 回だけ読み込むようにし、missing-file case には null sentinel を使います。15 件の regression tests で cache behavior、fs 呼び出し回数、injection guard、missing-file sentinel、cache reset をカバーしています。(Fixes #1202)
- **integration tests を modernize**。
- README の **install caveats を明確化**。

### Code Review Consolidation

`requesting-code-review` は自己完結型になりました。persona、checklist、dispatch template は `skills/requesting-code-review/code-reviewer.md` に集約され、skill は `Task (general-purpose)` を直接 dispatch します。(PR #1299)

- **単一の source of truth** — 以前は `agents/code-reviewer.md` と skill の placeholder template の両方にあり、別々に drift していた persona/checklist が 1 つの file に統合されました。
- **`subagent-driven-development` も同様に変更** — `code-quality-reviewer-prompt.md` は named agent ではなく `Task (general-purpose)` を dispatch するようになりました。
- **振る舞いテストを追加** — `tests/claude-code/test-requesting-code-review.sh` は小さな project に本物の bug（SQL injection、平文 password 処理、credential logging）を埋め込み、dispatch された reviewer が埋め込んだ全 issue を Critical/Important severity で指摘し、diff を approve しないことを検証します。
- **Codex と Copilot の workaround docs を整理** — `references/codex-tools.md` と `references/copilot-tools.md` にあった「Named agent dispatch」section は、named agent を generic dispatch に平坦化する方法を説明していました。出荷される named agent がなくなったため workaround は不要になり、両 section は削除されました。

### Subagent-Driven Development

- **3 task ごとに停止しない** — `requesting-code-review`（元は `executing-plans` 用）にあった「各 batch（3 task）後に review」という cadence が `subagent-driven-development` に漏れ込んでいました。これを「各 task または自然な checkpoint ごと」に置き換え、継続実行の明示的な指示を追加しました。
- **SDD integration test が実際に assertion を実行** — 3 つの独立した bug により、この test は verification 結果を出力する前に黙って終了していました。原因は、working-dir path の未解決 `..` segment、`set -euo pipefail` と `find | sort | head -1` の相互作用（producer 側の SIGPIPE で script が終了）、そして `claude -p` 呼び出しに `--plugin-dir` がなく、working tree ではなくインストール済み plugin を読み込んでいたことです。3 件すべて修正し、6 つの verification tests が実際の end-to-end SDD run に対して実行されるようになりました。

### Cursor

- **Windows の SessionStart hook** を、拡張子なしの `session-start` script を直接呼ぶ代わりに `run-hook.cmd` 経由に変更しました。これにより、Windows がその file を実行ではなく editor で開いてしまう問題を修正しました。加えて、`hooks-cursor.json` に紛れ込んでいた UTF-8 BOM も削除しました。

### Gemini CLI

- **subagent dispatch mapping** — Gemini の `Task` dispatch は `@agent-name` / `@generalist` に対応付けられ、独立した task に対する parallel subagent dispatch も文書化されました。

### Skills

- skill 内容全体で **terminology cleanup** を実施。

### Documentation & Install

- README に **Factory Droid の installation instructions** を追加。
- README に **Quickstart install links** を追加。(PR #1293 by @arittr)
- **Codex plugin の install guidance** を更新。(PR #1288 by @arittr)
- tools reference の **Codex `wait` mapping** を `wait_agent` に修正。
- **install 順序を再編成** し、Codex の install instructions を整理。
- 単一の source として `RELEASE-NOTES.md` を使うため、使われていなかった **`CHANGELOG.md` を削除**。(PR #1163 by @shaanmajid)
- **Discord 招待リンク** を修正し、Community section に release announcements へのリンクと詳しい Discord 説明を追加。

### Community

- @shaanmajid — 使われていなかった `CHANGELOG.md` の削除 (PR #1163)
- @arittr — README の quickstart install links (#1293)、Codex plugin の install guidance (#1288)、`sync-to-codex-plugin` の `interface.defaultPrompt` seed (#1180)

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI Support

- **SessionStart context injection** — Copilot CLI v1.0.11 で、sessionStart hook の出力に `additionalContext` を含めることがサポートされました。session-start hook は `COPILOT_CLI` environment variable を検出し、SDK 標準の `{ "additionalContext": "..." }` 形式を出力するようになりました。これにより、Copilot CLI ユーザーは session 開始時に完全な superpowers bootstrap を受け取れます。（元の修正は @culinablaz による PR #910）
- **tool mapping** — Claude Code から Copilot CLI への完全な tool 対応表をまとめた `references/copilot-tools.md` を追加
- **skill と README の更新** — `using-superpowers` skill の platform instructions と README の installation section に Copilot CLI を追加

### OpenCode Fixes

- **skills path の一貫性** — bootstrap text は、runtime path と一致しない紛らわしい `configDir/skills/superpowers/` path を案内しなくなりました。agent は path をたどって files を探すのではなく、native の `skill` tool を使うべきです。tests も単一の source of truth から導かれる一貫した path を使うようになりました。(#847, #916)
- **bootstrap を user message に変更** — bootstrap injection を `experimental.chat.system.transform` から `experimental.chat.messages.transform` へ移し、system message を追加する代わりに最初の user message の先頭へ付加するようにしました。これにより、毎 turn 繰り返される system messages による token 増加を避け（#750）、複数の system messages で壊れる Qwen などの model との互換性も修正しました。(#894)

## v5.0.6 (2026-03-24)

### Inline Self-Review が Subagent Review Loops を置き換え

subagent review loop（plan/spec を review するために新しい agent を dispatch する方式）は、plan quality を測定可能に改善しないまま実行時間を 2 倍（約 25 分の overhead）にしていました。5 つの version に対して各 5 回の trial を行った regression testing では、review loop の有無にかかわらず品質スコアは同一でした。

- **brainstorming** — Spec Review Loop（subagent dispatch + 3 iteration cap）を、placeholder scan、internal consistency、scope check、ambiguity check を含む inline Spec Self-Review checklist に置き換え
- **writing-plans** — Plan Review Loop（subagent dispatch + 3 iteration cap）を、spec coverage、placeholder scan、type consistency を含む inline Self-Review checklist に置き換え
- **writing-plans** — plan failure を定義する明示的な "No Placeholders" section を追加（TBD、曖昧な説明、未定義参照、"similar to Task N"）
- self-review は、subagent approach と同等の defect rate を維持しつつ、約 25 分ではなく約 30 秒で実行ごとに 3〜5 件の本物の bug を捕捉します

### Brainstorm Server

- **session directory を再構成** — brainstorm server の session directory は、`content/`（browser に配信する HTML files）と `state/`（events、server-info、pid、log）の 2 つの並列 subdirectory を持つようになりました。以前は server state と user interaction data が配信コンテンツと同じ場所に保存され、HTTP 経由でアクセス可能でした。`screen_dir` と `state_dir` の path は両方とも server-started JSON に含まれます。（報告: 吉田仁）

### Bug Fixes

- **owner-PID lifecycle fixes** — brainstorm server の owner-PID monitoring には、60 秒以内の誤 shutdown を引き起こす 2 つの bug がありました: (1) cross-user PID（Tailscale SSH など）からの EPERM を「process が死んだ」と扱っていたこと、(2) WSL では grandparent PID が最初の lifecycle check より前に終了する短命 subprocess に解決されること。EPERM を「生存」と扱い、起動時に owner PID を検証することで修正しました。すでに死んでいれば monitoring を無効化し、server は 30 分の idle timeout に依存します。これにより、server が一般化して処理できるようになったため、`start-server.sh` から Windows/MSYS2 固有の carve-out も削除されました。(#879)
- **writing-skills** — SKILL.md frontmatter が「2 つの fields のみをサポートする」とする誤った記述を修正し、「2 つの必須 fields」としたうえで、サポートされるすべての fields について agentskills.io specification へのリンクを追加しました (PR #882 by @arittr)

### Codex App Compatibility

- **codex-tools** — Claude Code の named agent types を worker roles 付き `spawn_agent` に変換する方法を記した named agent dispatch mapping を追加 (PR #647 by @arittr)
- **codex-tools** — worktree-aware skills 向けに環境検出と Codex App finishing sections を追加 (by @arittr)
- **Design spec** — read-only environment detection、worktree-safe skill behavior、sandbox fallback patterns を扱う Codex App compatibility design spec (PRI-823) を追加 (by @arittr)

## v5.0.5 (2026-03-17)

### Bug Fixes

- **Brainstorm server の ESM 修正** — `server.js` → `server.cjs` にリネームし、ルートの `package.json` の `"type": "module"` により `require()` が失敗していた Node.js 22+ でも brainstorming server が正しく起動するようにしました。(PR #784 by @sarbojitrana, fixes #774, #780, #783)
- **Windows での Brainstorm owner-PID** — Node.js から PID namespace が見えない Windows/MSYS2 では PID lifecycle monitoring をスキップし、server が 60 秒後に自己終了してしまうのを防ぎました。(#770, docs from PR #768 by @lucasyhzlu-debug)
- **stop-server.sh の信頼性** — 成功を報告する前に server process が実際に終了したことを検証するようにしました。SIGTERM + 2 秒待機 + SIGKILL fallback。(#723)

### 変更

- **実行 handoff** — plan 作成後、subagent-driven と inline execution のどちらにするかを再びユーザーが選べるように戻しました。subagent-driven を推奨しますが、必須ではなくなりました。

## v5.0.4 (2026-03-16)

### Review Loop の改善

不要な review pass を削減し、reviewer の focus を絞ることで、spec と plan の review に使う token を大幅に減らし、速度も向上させました。

- **plan 全体を 1 回で review** — plan reviewer は chunk ごとではなく、完全な plan を 1 回で review するようになりました。chunk 関連の概念（`## Chunk N:` 見出し、1000 行 chunk 制限、chunk ごとの dispatch）をすべて削除しました。
- **blocker とする issue の基準を引き上げ** — spec reviewer と plan reviewer の両 prompt に "Calibration" section を追加しました。implementation 時に実際の問題を引き起こす issue だけを指摘すべきであり、些細な wording、style の好み、formatting への細かな指摘は approval を block する理由にしません。
- **review iteration 上限を削減** — spec と plan の両 review loop で最大回数を 5 から 3 に減らしました。reviewer が正しく calibration されていれば、3 回で十分です。
- **reviewer checklist を簡素化** — spec reviewer は 7 category から 5 へ、plan reviewer は 7 から 4 へ絞りました。formatting 重視の check（task syntax、chunk size）を削除し、実質的な内容（buildability、spec との整合性）を優先しています。

### OpenCode

- **1 行で plugin install** — OpenCode plugin は `config` hook 経由で skills directory を自動登録するようになりました。symlink や `skills.paths` config は不要です。install は `opencode.json` に 1 行追加するだけです。(PR #753)
- git から npm package として superpowers を OpenCode が install できるよう、**`package.json` を追加**しました。

### Bug Fixes

- **server が本当に停止したことを確認** — `stop-server.sh` は成功を報告する前に process が死んでいることを確認するようになりました。SIGTERM + 2 秒待機 + SIGKILL fallback。生き残っていれば失敗を報告します。(PR #751)
- **汎用的な agent 表現** — brainstorm companion の待機ページにある文言を "Claude" ではなく "the agent" に変更しました。

## v5.0.3 (2026-03-15)

### Cursor Support

- **Cursor hooks** — Cursor の camelCase 形式（`sessionStart`, `version: 1`）に合わせた `hooks/hooks-cursor.json` を追加し、`.cursor-plugin/plugin.json` から参照するよう更新しました。`session-start` の platform detection も `CURSOR_PLUGIN_ROOT` を先に確認するよう修正しました（Cursor は `CLAUDE_PLUGIN_ROOT` も設定することがあります）。(Based on PR #709)

### Bug Fixes

- **`--resume` で SessionStart hook が再発火しないように修正** — resume した session は会話履歴内にすでに context を持っているため、startup hook が context を再注入していました。hook は `startup`、`clear`、`compact` のときだけ発火します。
- **Bash 5.3+ で hook が hang する問題** — `hooks/session-start` で heredoc（`cat <<EOF`）を `printf` に置き換えました。heredoc 内で大きな変数展開を行う bash regression により、macOS の Homebrew bash 5.3+ で無期限に hang する問題を修正します。(#572, #571)
- **POSIX-safe な hook script** — `hooks/session-start` で `${BASH_SOURCE[0]:-$0}` を `$0` に置き換えました。`/bin/sh` が dash の Ubuntu/Debian で起きていた "Bad substitution" error を修正します。(#553)
- **移植性の高い shebang** — すべての shell script の `#!/bin/bash` を `#!/usr/bin/env bash` に置き換えました。`/bin/bash` が古い、または存在しない NixOS、FreeBSD、Homebrew bash の macOS での実行を修正します。(#700)
- **Windows での Brainstorm server** — Windows/Git Bash（`OSTYPE=msys*`, `MSYSTEM`）を自動検出して foreground mode に切り替えるようにし、`nohup` / `disown` による process reaping が原因の server 無言失敗を修正しました。(#737)
- **Codex docs 修正** — Codex documentation 内の非推奨 `collab` flag を `multi_agent` に置き換えました。(PR #749)

## v5.0.2 (2026-03-11)

### Zero-Dependency Brainstorm Server

**vendored な node_modules をすべて削除 — server.js は完全に自己完結型になりました**

- Express/Chokidar/WebSocket 依存を、組み込みの `http`、`fs`、`crypto` module だけを使う zero-dependency Node.js server に置き換え
- 約 1,200 行分の vendored `node_modules/`、`package.json`、`package-lock.json` を削除
- Custom WebSocket protocol 実装（RFC 6455 framing、ping/pong、適切な close handshake）
- Chokidar の代わりに native `fs.watch()` による file watching を使用
- HTTP serving、WebSocket protocol、file watching、integration tests を含む完全な test suite

### Brainstorm Server Reliability

- **30 分 idle で自動終了** — client が接続していない場合に server を shutdown し、孤児 process を防止
- **owner process tracking** — 親 harness の PID を監視し、所有 session が死んだら終了
- **liveness check** — 既存 instance を再利用する前に、skill が server の応答性を確認
- **encoding 修正** — 配信 HTML page に適切な `<meta charset="utf-8">` を追加

### Subagent Context Isolation

- すべての delegation skill（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）に context isolation principle を追加
- subagent には必要な context だけを渡し、context window の汚染を防止

## v5.0.1 (2026-03-10)

### Agentskills Compliance

**Brainstorm-server を skill directory 内へ移動**

- [agentskills.io](https://agentskills.io) specification に従い、`lib/brainstorm-server/` → `skills/brainstorming/scripts/` へ移動
- `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` への参照をすべて相対 `scripts/` path に置き換え
- これにより skills は platform をまたいで完全に portable になり、script の場所を特定するための platform 固有 env var が不要に
- `lib/` directory を削除（最後に残っていた内容でした）

### New Features

**Gemini CLI extension**

- `gemini-extension.json` と repo root の `GEMINI.md` により、native Gemini CLI extension support を追加
- `GEMINI.md` は session start 時に `using-superpowers` skill と tool mapping table を `@import`
- Gemini CLI tool mapping reference（`skills/using-superpowers/references/gemini-tools.md`）を追加 — Claude Code の tool 名（Read、Write、Edit、Bash など）を Gemini CLI の同等機能（read_file、write_file、replace など）に変換
- Gemini CLI の制限事項も文書化 — subagent support がないため、skills は `executing-plans` に fallback
- cross-platform compatibility のため extension root を repo root に配置（Windows の symlink 問題を回避）
- install instructions を README に追加

### Improvements

**Multi-platform brainstorm server 起動**

- visual-companion.md に platform ごとの launch instructions を追加: Claude Code（default mode）、Codex（`CODEX_CI` による auto-foreground）、Gemini CLI（`is_background` と `--foreground`）、およびその他環境向け fallback
- background execution で stdout が隠れていても agent が URL と port を見つけられるよう、server は startup JSON を `$SCREEN_DIR/.server-info` に書き出すようになりました

**Brainstorm server dependencies を同梱**

- fresh plugin install 直後でも runtime に `npm` を要求せず brainstorm server が動くよう、`node_modules` を repo に vendored しました
- bundled deps から `fsevents` を削除（macOS 専用 native binary。なくても chokidar は問題なく fallback）
- `node_modules` がない場合は `npm install` で自動 install する fallback を追加

**OpenCode tool mapping 修正**

- `TodoWrite` → `todowrite`（誤って `update_plan` に mapping されていたものを修正）。OpenCode source に照らして確認済みです

### Bug Fixes

**Windows/Linux: single quotes が SessionStart hook を壊す** (#577, #529, #644, PR #585)

- hooks.json で `${CLAUDE_PLUGIN_ROOT}` を single quotes で囲むと、Windows では cmd.exe が single quotes を path delimiter として認識せず、Linux では variable expansion が起きないため失敗していました
- 修正: single quotes を escape した double quotes に置き換え — path に空白がある場合を含め、macOS bash、Windows cmd.exe、Windows Git Bash、Linux のすべてで動作します
- Windows 11 (NT 10.0.26200.0) + Claude Code 2.1.72 + Git for Windows で検証済み

**Brainstorming の spec review loop が飛ばされていた** (#677)

- spec review loop（spec-document-reviewer subagent を dispatch し、approve されるまで反復する処理）は prose の "After the Design" section にだけ存在し、checklist と process flow diagram から抜けていました
- agent は prose より diagram と checklist に従う傾向が強いため、この spec review step は完全に飛ばされていました
- checklist に step 7（spec review loop）を追加し、対応する node も dot graph に追加しました
- `claude --plugin-dir` と `claude-session-driver` で test し、worker が reviewer を正しく dispatch することを確認しました

**Cursor install command** (PR #676)

- README の Cursor install command を修正: `/plugin-add` → `/add-plugin`（Cursor 2.5 release announcement で確認）

**brainstorming の user review gate** (#565)

- spec 完成後から writing-plans へ handoff する前の間に、明示的な user review step を追加
- implementation planning を始める前に、ユーザーの spec 承認が必須に
- checklist、process flow、prose をこの新しい gate に合わせて更新

**session-start hook が platform ごとに context を 1 回だけ出力**

- hook が Claude Code 上で動いているか他 platform 上かを検出するように変更
- Claude Code では `hookSpecificOutput`、それ以外では `additional_context` を出力 — 二重の context injection を防止します

**token analysis script の linting 修正**

- `tests/claude-code/analyze-token-usage.py` 内の `except:` → `except Exception:`

### Maintenance

**dead code を削除**

- `lib/skills-core.js` とその test（`tests/opencode/test-skills-core.js`）を削除 — 2026 年 2 月以降未使用でした
- `tests/opencode/test-plugin-loading.sh` から skills-core の existence check を削除

### Community

- @karuturi — Claude Code official marketplace install instructions (PR #610)
- @mvanhorn — session-start hook dual-emit fix, OpenCode tool mapping fix
- @daniel-graham — bare except に対する linting fix
- PR #585 author — Windows/Linux hooks quoting fix

---

## v5.0.0 (2026-03-09)

### Breaking Changes

**specs と plans の directory 構成を再編**

- Specs（brainstorming の出力）は `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` に保存されるようになりました
- Plans（writing-plans の出力）は `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md` に保存されるようになりました
- spec/plan の保存場所に関する user preference がある場合は、この default を上書きします
- すべての内部 skill 参照、test file、example path を新しい構成に合わせて更新
- Migration: 必要なら既存 file を `docs/plans/` から新しい場所へ移動してください

**subagent-driven development が capable な harness では必須に**

writing-plans は、subagent-driven-development と executing-plans のどちらを使うか選ばせなくなりました。subagent support のある harness（Claude Code、Codex）では subagent-driven-development が必須です。executing-plans は subagent capability のない harness 向けに限定され、Superpowers は subagent 対応 platform のほうがより効果的に動作することをユーザーへ伝えるようになりました。

**executing-plans が batch 実行しなくなりました**

"3 task 実行してから review のため停止" という pattern を削除しました。plan は blocker が出るまで継続的に実行されます。

**slash commands を非推奨化**

`/brainstorm`、`/write-plan`、`/execute-plan` は、対応する skill を使うよう案内する deprecation notice を表示するようになりました。これらの command は次の major release で削除されます。

### New Features

**visual brainstorming companion**

brainstorming session 向けの、任意利用の browser-based companion です。topic が visual を必要とする場合、brainstorming skill は mockup、diagram、comparison、その他の content を terminal conversation と並行して browser window に表示することを提案します。

- `lib/brainstorm-server/` — browser helper library、session management scripts、dark/light theme 対応 frame template（"Superpowers Brainstorming" と GitHub link 付き）を備えた WebSocket server
- `skills/brainstorming/visual-companion.md` — server workflow、screen authoring、feedback collection を段階的に開示する guide
- Brainstorming skill の process flow に visual companion の判断ポイントを追加: project context を探索した後、その後の質問が visual content を含むかを評価し、別 message で companion を提案します
- question ごとの判断: いったん受け入れた後も、各質問について browser と terminal のどちらが適切かを評価
- `tests/brainstorm-server/` に integration tests を追加

**document review system**

subagent dispatch を使う、spec document と plan document の自動 review loop:

- `skills/brainstorming/spec-document-reviewer-prompt.md` — completeness、consistency、architecture、YAGNI を reviewer が確認
- `skills/writing-plans/plan-document-reviewer-prompt.md` — spec との整合性、task 分解、file structure、file size を reviewer が確認
- Brainstorming は design doc 作成後に spec reviewer を dispatch
- Writing-plans は各 section の後に chunk-based plan review loop を実行
- review loop は approve されるか、5 iteration 後に escalate するまで繰り返し
- `tests/claude-code/test-document-review-system.sh` に end-to-end tests を追加
- design spec と implementation plan は `docs/superpowers/` に配置

**skill pipeline 全体への architecture guidance**

brainstorming、writing-plans、subagent-driven-development に、design-for-isolation と file-size-awareness の guidance を追加:

- **Brainstorming** — 新 section: "Design for isolation and clarity"（明確な境界、よく定義された interface、独立して test 可能な unit）と "Working in existing codebases"（既存 pattern に従い、改善は限定的に行う）
- **Writing-plans** — 新しい "File Structure" section: task を定義する前に file と責務を整理。新しい "Scope Check" backstop: 本来 brainstorming 中に分解されるべき multi-subsystem spec を捕捉
- **SDD implementer** — 新しい "Code Organization" section（plan の file structure に従う、file が肥大化しそうなら懸念を報告）と、"When You're in Over Your Head" escalation guidance
- **SDD code quality reviewer** — architecture、unit 分解、plan 準拠、file growth を check するように変更
- **Spec/plan reviewers** — review criteria に architecture と file size を追加
- **scope assessment** — Brainstorming が、project が単一 spec には大きすぎないかを評価するようになりました。multi-subsystem の request は早い段階で flag され、spec → plan → implementation cycle を個別に持つ sub-project に分解されます

**subagent-driven development の改善**

- **model selection** — task 種別ごとに model capability を選ぶ guidance を追加: 機械的な implementation には cheap model、integration には standard、architecture と review には高性能 model
- **implementer status protocol** — subagent は DONE、DONE_WITH_CONCERNS、BLOCKED、NEEDS_CONTEXT を報告するように。controller はそれぞれに応じて、より多い context で再 dispatch、model capability の引き上げ、task の分割、または human への escalation を行います

### Improvements

**instruction priority hierarchy**

using-superpowers に、明示的な priority ordering を追加しました:

1. ユーザーの明示的 instructions（CLAUDE.md、AGENTS.md、直接の依頼）— 最優先
2. Superpowers skills — default の system behavior を上書き
3. Default system prompt — 最も低い優先度

CLAUDE.md や AGENTS.md に "don't use TDD" と書かれていて、skill に "always use TDD" と書かれている場合は、ユーザーの instruction が優先されます。

**SUBAGENT-STOP gate**

using-superpowers に `<SUBAGENT-STOP>` block を追加しました。特定 task のために dispatch された subagent は、その skill を起動する代わりに skip し、1% ルールや完全な skill workflow の起動を避けます。

**multi-platform の改善**

- Codex tool mapping を progressive disclosure な reference file（`references/codex-tools.md`）へ移動
- non-Claude-Code platform が tool の同等機能を見つけられるよう、Platform Adaptation への案内を追加
- plan header が "Claude" ではなく "agentic workers" を対象にするよう変更
- `docs/README.codex.md` に collab feature requirement を記載

**writing-plans template 更新**

- 進捗 tracking のため、plan step が checkbox syntax（`- [ ] **Step N:**`）を使うようになりました
- plan header が、platform-aware routing を踏まえて subagent-driven-development と executing-plans の両方を参照するようになりました

---

## v4.3.1 (2026-02-21)

### Added

**Cursor support**

Superpowers が Cursor の plugin system で動作するようになりました。`.cursor-plugin/plugin.json` manifest と、README の Cursor 専用 install instructions を含みます。SessionStart hook の出力には、Cursor hook 互換のため、既存の `hookSpecificOutput.additionalContext` に加えて `additional_context` field も含まれるようになりました。

### Fixed

**Windows: 信頼性の高い hook 実行のため polyglot wrapper を復元 (#518, #504, #491, #487, #466, #440)**

Claude Code の Windows における `.sh` 自動検出が hook command の先頭に `bash` を付けるようになり、実行が壊れていました。修正内容:

- `session-start.sh` を拡張子なしの `session-start` にリネームし、自動検出が干渉しないように変更
- `run-hook.cmd` polyglot wrapper を復元し、複数候補から bash を検出（標準的な Git for Windows path、次に PATH fallback）
- bash が見つからない場合は error にせず黙って終了
- Unix では wrapper が `exec bash` 経由で script を直接実行
- POSIX-safe な `dirname "$0"` path 解決を使用（bash だけでなく dash/sh でも動作）

これにより、path に空白がある Windows、WSL がない環境、MSYS 上での `set -euo pipefail` の脆さ、backslash の破損といった SessionStart failure を修正しました。

## v4.3.0 (2026-02-12)

この修正により、superpowers skills の compliance が大幅に改善され、Claude が意図せず native plan mode に入ってしまう可能性も減るはずです。

### Changed

**Brainstorming skill が workflow を「説明」するのではなく「強制」するように**

model が design phase を飛ばして frontend-design のような implementation skill に直行したり、brainstorming の全工程を 1 つの text block に潰してしまうことがありました。これを防ぐため、skill は hard gate、mandatory checklist、graphviz process flow によって compliance を強制するようになりました:

- `<HARD-GATE>`: design を提示して user が approve するまでは、implementation skill、code、scaffolding を禁止
- task として作成し、順に完了させるべき明示的 checklist（6 項目）
- 唯一の有効な terminal state を `writing-plans` とする Graphviz process flow
- "this is too simple to need a design" という anti-pattern を明示 — これは model が process を飛ばす際によく使う rationalization そのものです
- design section のサイズを、project の複雑さではなく section の複雑さに基づいて決定

**Using-superpowers の workflow graph が EnterPlanMode を intercept**

skill flow graph に `EnterPlanMode` intercept を追加しました。model が Claude の native plan mode に入ろうとしたとき、brainstorming が済んでいるかを確認し、未実施なら brainstorming skill を経由させます。plan mode 自体には入りません。

### Fixed

**SessionStart hook が同期実行されるように**

hooks.json の `async: true` を `async: false` に変更しました。async だと hook が model の最初の turn までに完了せず、最初の message では using-superpowers instructions が context に入っていない可能性がありました。

## v4.2.0 (2026-02-05)

### Breaking Changes

**Codex: bootstrap CLI を native skill discovery に置き換え**

`superpowers-codex` bootstrap CLI、Windows 用 `.cmd` wrapper、関連する bootstrap content file を削除しました。Codex は `~/.agents/skills/superpowers/` symlink による native skill discovery を使うようになったため、従来の `use_skill` / `find_skills` CLI tool は不要になりました。

install は clone + symlink だけになり（INSTALL.md に記載）、Node.js dependency も不要です。以前の `~/.codex/skills/` path は非推奨になりました。

### Fixes

**Windows: Claude Code 2.1.x の hook 実行を修正 (#331)**

Claude Code 2.1.x は Windows 上での hook 実行方法を変更し、command 内の `.sh` file を自動検出して先頭に `bash` を付けるようになりました。これにより polyglot wrapper pattern が壊れ、`bash "run-hook.cmd" session-start.sh` が `.cmd` file を bash script として実行しようとしていました。

修正: hooks.json は `session-start.sh` を直接呼ぶようになりました。Claude Code 2.1.x が bash invocation を自動で処理します。加えて、Windows checkout 時の CRLF 問題を防ぐため、shell script に LF line ending を強制する `.gitattributes` も追加しました。

**Windows: terminal freeze を防ぐため SessionStart hook を async 実行 (#404, #413, #414, #419)**

同期実行の SessionStart hook が TUI の raw mode 移行を block し、すべての keyboard input が freeze していました。hook を async で動かすことで、superpowers context を注入しつつ freeze を防ぎます。

**Windows: `escape_for_json` の O(n^2) performance を修正**

`${input:$i:1}` を使う 1 文字ずつの loop は、substring copy overhead のため bash では O(n^2) になっていました。Windows Git Bash ではこれに 60 秒以上かかっていました。これを bash parameter substitution（`${s//old/new}`）に置き換え、各 pattern を C レベルの単一 pass で処理するようにした結果、macOS で 7 倍高速化し、Windows ではさらに大幅に改善しました。

**Codex: Windows/PowerShell invocation を修正 (#285, #243)**

- Windows は shebang を尊重しないため、拡張子なしの `superpowers-codex` script を直接呼ぶと "Open with" dialog が出ていました。すべての invocation に `node` を前置するように変更しました。
- Windows での `~/` path expansion を修正 — PowerShell は `node` への引数として渡された `~` を展開しません。代わりに bash と PowerShell の両方で正しく展開される `$HOME` を使うように変更しました。

**Codex: installer の path resolution を修正**

手書きの URL pathname parsing ではなく `fileURLToPath()` を使うようにし、空白や特殊文字を含む path を全 platform で正しく処理できるようにしました。

**Codex: writing-skills 内の古い skills path を修正**

非推奨の `~/.codex/skills/` 参照を、native discovery 用の `~/.agents/skills/` に更新しました。

### Improvements

**implementation 前に worktree isolation が必須に**

`subagent-driven-development` と `executing-plans` の両方にとって、`using-git-worktrees` が必須 skill になりました。implementation workflow では、作業開始前に分離された worktree のセットアップを明示的に要求し、main 上で直接作業してしまう事故を防ぎます。

**main branch 保護を緩和し、明示的な同意があれば許可**

main branch での作業を全面禁止する代わりに、明示的な user consent があれば許可するようにしました。柔軟性を高めつつ、影響をユーザーが認識したうえで進められるようにしています。

**install 検証を簡素化**

検証手順から `/help` command check と、特定 slash command の一覧を削除しました。skills は主に、特定 command を実行するのではなく、やりたいことを説明することで呼び出されるためです。

**Codex: bootstrap 内の subagent tool mapping を明確化**

subagent workflow において、Codex の tool が Claude Code の同等機能へどう対応するかの documentation を改善しました。

### Tests

- subagent-driven-development 向けの worktree requirement test を追加
- main branch red flag warning test を追加
- skill recognition test assertions の case sensitivity を修正

---

## v4.1.1 (2026-01-23)

### Fixes

**OpenCode: 公式 docs に合わせて `plugins/` directory に統一 (#343)**

OpenCode の公式 documentation は `~/.config/opencode/plugins/`（複数形）を使用しています。これまでの docs では `plugin/`（単数形）を使っていました。OpenCode 自体は両方受け入れますが、混乱を避けるため公式 convention に統一しました。

変更内容:
- repo 構成内の `.opencode/plugin/` を `.opencode/plugins/` にリネーム
- 全 platform の installation docs（INSTALL.md、README.opencode.md）を更新
- test script を対応させて更新

**OpenCode: symlink instructions を修正 (#339, #342)**

- 再 install 時の "file already exists" error を防ぐため、`ln -s` の前に明示的な `rm` を追加
- INSTALL.md に抜けていた skills symlink 手順を追加
- 非推奨の `use_skill` / `find_skills` 参照を、native `skill` tool 参照に更新

---

## v4.1.0 (2026-01-23)

### Breaking Changes

**OpenCode: native skills system へ移行**

OpenCode 向け Superpowers は、独自の `use_skill` / `find_skills` tool ではなく、OpenCode 標準の `skill` tool を使うようになりました。OpenCode 組み込みの skill discovery と連携する、よりクリーンな integration です。

**Migration required:** Skills は `~/.config/opencode/skills/superpowers/` に symlink する必要があります（更新された installation docs を参照）。

### Fixes

**OpenCode: session start 時の agent reset を修正 (#226)**

以前の bootstrap injection は `session.prompt({ noReply: true })` を使っていたため、最初の message で選択済み agent が "build" に reset されていました。現在は副作用なく system prompt を直接変更する `experimental.chat.system.transform` hook を使用します。

**OpenCode: Windows installation を修正 (#232)**

- `skills-core.js` への dependency を削除（file を symlink ではなく copy した場合に壊れていた相対 import を排除）
- cmd.exe、PowerShell、Git Bash 向けの包括的な Windows installation docs を追加
- 各 platform に適した symlink / junction の使い分けを文書化

**Claude Code: Claude Code 2.1.x の Windows hook 実行を修正**

Claude Code 2.1.x は Windows 上での hook 実行方法を変更し、command 中の `.sh` file を自動検出して先頭に `bash ` を付けるようになりました。これにより polyglot wrapper pattern が壊れ、`bash "run-hook.cmd" session-start.sh` が .cmd file を bash script として実行しようとしていました。

修正: hooks.json は `session-start.sh` を直接呼ぶようになりました。Claude Code 2.1.x が bash invocation を自動で処理します。さらに、Windows checkout 時の CRLF 問題を防ぐため shell script に LF line ending を強制する `.gitattributes` も追加しました。

---

## v4.0.3 (2025-12-26)

### Improvements

**using-superpowers skill を、明示的な skill request に強く対応するよう改善**

ユーザーが skill 名を明示して依頼したとき（例: "subagent-driven-development, please"）でも Claude が skill の呼び出しを飛ばしてしまう failure mode に対処しました。Claude は "意味はわかる" と判断し、skill を読み込まずに直接作業を始めていました。

変更内容:
- "The Rule" を "Check for skills" から "Invoke relevant or requested skills" に変更 — 受動的な確認ではなく、積極的な起動を強調
- "BEFORE any response or action" を追加 — もともとの wording は "response" にしか触れておらず、Claude が先に action を取ることがありました
- 間違った skill を呼んでも問題ないという reassurance を追加 — ためらいを減らします
- 新しい red flag を追加: "I know what that means" → 概念を知っていることと skill を使うことは別です

**明示的な skill request の test を追加**

`tests/explicit-skill-requests/` に新しい test suite を追加し、ユーザーが skill 名を指定したときに Claude が正しく skill を呼ぶことを検証します。single-turn と multi-turn の test scenario を含みます。

## v4.0.2 (2025-12-23)

### Fixes

**slash commands はユーザー専用に**

3 つの slash command（`/brainstorm`、`/execute-plan`、`/write-plan`）すべてに `disable-model-invocation: true` を追加しました。Claude は Skill tool 経由でこれらの command を呼び出せなくなり、手動の user invocation のみに制限されます。

基盤となる skills（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）は引き続き Claude が自律的に呼び出せます。この変更は、結局 skill へリダイレクトするだけの command を Claude が呼んでしまう混乱を防ぎます。

## v4.0.1 (2025-12-23)

### Fixes

**Claude Code で skills にアクセスする方法を明確化**

Claude が Skill tool で skill を起動したあと、その skill file をさらに Read しようとする混乱した pattern を修正しました。`using-superpowers` skill で、Skill tool は skill content を直接読み込むため、別途 file を読む必要はないと明示しています。

- `using-superpowers` に "How to Access Skills" section を追加
- instruction 内の "read the skill" を "invoke the skill" に変更
- slash commands を fully qualified な skill 名（例: `superpowers:brainstorming`）を使う形に更新

**receiving-code-review に GitHub thread reply guidance を追加** (h/t @ralphbean)

inline review comment には top-level PR comment としてではなく、元の thread 内で reply するようにという note を追加しました。

**writing-skills に automation-over-documentation guidance を追加** (h/t @EthanJStark)

機械的な制約は documentation ではなく automation で解決すべきであり、skills は判断が必要な場面に使うべきだという guidance を追加しました。

## v4.0.0 (2025-12-17)

### New Features

**subagent-driven-development に 2 段階の code review を導入**

subagent workflow は、各 task の後に 2 つの独立した review stage を使うようになりました:

1. **spec compliance review** - 懐疑的な reviewer が implementation が spec と完全一致しているかを検証します。欠けている requirement だけでなく、作り込み過ぎも検出します。implementer の報告を信用せず、実際の code を読みます。

2. **code quality review** - spec compliance が通ったあとにのみ実行されます。clean code、test coverage、maintainability を review します。

これにより、code 自体はきれいだが要求内容と一致していない、という典型的な failure mode を捕捉できます。review は one-shot ではなく loop です。reviewer が issue を見つけた場合、implementer が修正し、その後 reviewer が再確認します。

その他の subagent workflow 改善:
- controller が worker に file 参照ではなく task の全文を渡すように
- worker は作業前だけでなく作業中にも clarifying question を出せるように
- completion を報告する前に self-review checklist を実施
- plan は開始時に 1 回だけ読んで TodoWrite に抽出

`skills/subagent-driven-development/` に新しい prompt template を追加:
- `implementer-prompt.md` - self-review checklist を含み、質問を促す
- `spec-reviewer-prompt.md` - requirement に対して懐疑的に検証
- `code-quality-reviewer-prompt.md` - 標準的な code review

**debugging technique を tools と統合**

`systematic-debugging` が、関連する technique と tool をまとめて同梱するようになりました:
- `root-cause-tracing.md` - bug を call stack に沿って逆方向にたどる
- `defense-in-depth.md` - 複数 layer に validation を追加
- `condition-based-waiting.md` - 任意の timeout を condition polling に置き換える
- `find-polluter.sh` - どの test が pollution を作っているかを二分探索で見つける script
- `condition-based-waiting-example.ts` - 実際の debugging session から得た完全な実装例

**testing anti-patterns reference**

`test-driven-development` に `testing-anti-patterns.md` を追加し、以下を扱います:
- 実際の振る舞いではなく mock の振る舞いを test する
- production class に test 専用 method を追加する
- dependency を理解しないまま mock を使う
- 構造上の前提を隠してしまう不完全な mock

**skill test infrastructure**

skill の振る舞いを検証するための 3 つの新しい test framework:

`tests/skill-triggering/` - skill 名を明示しない素朴な prompt からでも skill が trigger されるかを検証。description だけで十分に起動できるよう、6 つの skill を test します。

`tests/claude-code/` - `claude -p` を使う headless integration tests。session transcript（JSONL）解析により skill 使用を検証します。cost tracking 用の `analyze-token-usage.py` も含みます。

`tests/subagent-driven-dev/` - 2 つの完全な test project を使った end-to-end workflow 検証:
- `go-fractals/` - Sierpinski/Mandelbrot を扱う CLI tool（10 tasks）
- `svelte-todo/` - localStorage と Playwright を備えた CRUD app（12 tasks）

### Major Changes

**実行可能仕様としての DOT flowchart**

主要な skill を、DOT/GraphViz flowchart を権威ある process 定義として使う形に書き直しました。prose は補助的な内容になります。

**The Description Trap**（`writing-skills` に記載）: description に workflow の要約が含まれていると、flowchart の内容より description が優先されることを発見しました。Claude は詳細な flowchart を読む代わりに短い description に従ってしまいます。対策として、description は process 詳細を含まない trigger 専用（"Use when X"）でなければなりません。

**using-superpowers における skill priority**

複数の skill が適用可能な場合、process skill（brainstorming、debugging）が implementation skill より明示的に先に来るようになりました。"Build X" はまず brainstorming を起動し、その後に domain skill を使います。

**brainstorming trigger を強化**

description を命令形に変更: "You MUST use this before any creative work—creating features, building components, adding functionality, or modifying behavior."

### Breaking Changes

**skill の統合** - 6 つの standalone skill を統合:
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → `systematic-debugging/` に同梱
- `testing-skills-with-subagents` → `writing-skills/` に同梱
- `testing-anti-patterns` → `test-driven-development/` に同梱
- `sharing-skills` を削除（obsolete）

### Other Improvements

- **render-graphs.js** - skill から DOT diagram を抽出し、SVG に render する tool
- **using-superpowers 内の Rationalizations table** - "I need more context first"、"Let me explore first"、"This feels productive" などの新項目を含む、一覧しやすい形式
- **docs/testing.md** - Claude Code integration tests を使って skills を test するための guide

---

## v3.6.2 (2025-12-03)

### Fixed

- **Linux 互換性**: polyglot hook wrapper（`run-hook.cmd`）を POSIX 準拠の構文に修正
  - 16 行目の bash 固有 `${BASH_SOURCE[0]:-$0}` を標準的な `$0` に置き換え
  - `/bin/sh` が dash の Ubuntu/Debian で発生していた "Bad substitution" error を解消
  - Fixes #141

---

## v3.5.1 (2025-11-24)

### Changed

- **OpenCode Bootstrap Refactor**: bootstrap injection を `chat.message` hook から `session.created` event に切り替え
  - Bootstrap は `session.prompt()` と `noReply: true` を使って session 作成時に注入されるように
  - 冗長な skill loading を防ぐため、using-superpowers はすでに読み込まれていることを model に明示
  - bootstrap content の生成を共有 `getBootstrapContent()` helper に統合
  - よりクリーンな単一実装アプローチ（fallback pattern を削除）

---

## v3.5.0 (2025-11-23)

### Added

- **OpenCode Support**: OpenCode.ai 向け native JavaScript plugin
  - Custom tools: `use_skill` と `find_skills`
  - context compaction をまたいで skill を維持するための message insertion pattern
  - chat.message hook による自動 context injection
  - session.compacted event 時の自動再注入
  - 3 層の skill priority: project > personal > superpowers
  - project-local skills support（`.opencode/skills/`）
  - Codex との code reuse のための共有 core module（`lib/skills-core.js`）
  - 適切な isolation を備えた自動 test suite（`tests/opencode/`）
  - platform 固有 documentation（`docs/README.opencode.md`, `docs/README.codex.md`）

### Changed

- **Codex 実装をリファクタ**: 共有 `lib/skills-core.js` ES module を使うように変更
  - Codex と OpenCode 間の code duplication を解消
  - skill discovery と parsing の単一 source of truth を実現
  - Node.js interop により Codex が ES modules を正しく読み込めることを確認

- **Documentation を改善**: README を問題/解決策が明確に伝わるように全面的に書き直し
  - 重複 section と矛盾する情報を削除
  - 完全な workflow 説明（brainstorm → plan → execute → finish）を追加
  - platform ごとの install instructions を簡素化
  - 自動起動の主張より、skill-checking protocol を重視

---

## v3.4.1 (2025-10-31)

### Improvements

- superpowers bootstrap を最適化し、重複した skill 実行を排除しました。`using-superpowers` skill の内容は session context に直接提供されるようになり、他の skills に対してだけ Skill tool を使うよう明確に案内します。これにより overhead を減らし、session start で内容を受け取っているにもかかわらず agent が `using-superpowers` を手動実行してしまう混乱した loop を防ぎます。

## v3.4.0 (2025-10-30)

### Improvements

- `brainstorming` skill を簡素化し、元の会話中心の vision に戻しました。重厚な 6 phase process と形式的な checklist を削除し、自然な対話へ戻しています: 質問は 1 回に 1 つずつ行い、その後 200〜300 語の section で design を提示して確認を取ります。documentation と implementation handoff の機能は維持しています。

## v3.3.1 (2025-10-28)

### Improvements

- `brainstorming` skill を更新し、質問前の自律的な recon、recommendation 主導の意思決定、人間に優先順位付けを丸投げしないことを必須にしました。
- Strunk の "Elements of Style" の原則（不要語の削除、否定形から肯定形への変換、並列構造の改善）に沿って、`brainstorming` skill に writing clarity の改善を適用しました。

### Bug Fixes

- `writing-skills` の guidance を明確化し、agent ごとの正しい personal skill directory（Claude Code なら `~/.claude/skills`、Codex なら `~/.codex/skills`）を指すようにしました。

## v3.3.0 (2025-10-28)

### New Features

**Experimental Codex Support**
- `bootstrap` / `use-skill` / `find-skills` command を備えた統合 `superpowers-codex` script を追加
- cross-platform な Node.js 実装（Windows、macOS、Linux で動作）
- namespace 付き skills: superpowers の skill は `superpowers:skill-name`、personal は `skill-name`
- 同名の場合は personal skill が superpowers skill を上書き
- 見やすい skill 表示: 生の frontmatter を出さず、name/description のみ表示
- 補助的な context を表示: 各 skill の supporting files directory を提示
- Codex 向け tool mapping: TodoWrite→update_plan、subagent→manual fallback など
- 自動起動のための最小限の AGENTS.md を使った bootstrap integration
- Codex 専用の完全な install guide と bootstrap instructions

**Claude Code integration との主な違い:**
- 複数 script ではなく、単一の統合 script
- Codex 固有の同等機能へ置き換える tool substitution system
- subagent 処理を簡素化（delegation ではなく manual work）
- 用語を更新: "Core skills" ではなく "Superpowers skills"

### Files Added
- `.codex/INSTALL.md` - Codex users 向け install guide
- `.codex/superpowers-bootstrap.md` - Codex 向け調整を含む bootstrap instructions
- `.codex/superpowers-codex` - 全機能を備えた統合 Node.js executable

**Note:** Codex support は experimental です。この integration は core な superpowers functionality を提供しますが、user feedback に応じて今後 refinement が必要になる可能性があります。

## v3.2.3 (2025-10-23)

### Improvements

**using-superpowers skill が Read tool ではなく Skill tool を使うよう更新**
- skill の呼び出し instructions を Read tool から Skill tool に変更
- description を "using Read tool" → "using Skill tool" に更新
- step 3 を "Use the Read tool" → "Use the Skill tool to read and run" に更新
- rationalization list の "Read the current version" → "Run the current version" に更新

Skill tool は Claude Code で skill を呼び出すための正しい仕組みです。この更新により、agent を正しい tool へ導くよう bootstrap instructions を修正しました。

### Files Changed
- Updated: `skills/using-superpowers/SKILL.md` - Read から Skill への tool 参照変更

## v3.2.2 (2025-10-21)

### Improvements

**agent の rationalization に対抗するよう using-superpowers skill を強化**
- mandatory な skill checking について絶対的な表現を使う EXTREMELY-IMPORTANT block を追加
  - "If even 1% chance a skill applies, you MUST read it"
  - "You do not have a choice. You cannot rationalize your way out."
- MANDATORY FIRST RESPONSE PROTOCOL checklist を追加
  - agent が response 前に必ず完了すべき 5 step process
  - "responding without this = failure" という明示的な consequence を追加
- 8 つの具体的な回避 pattern を含む Common Rationalizations section を追加
  - "This is just a simple question" → WRONG
  - "I can check files quickly" → WRONG
  - "Let me gather information first" → WRONG
  - そのほか、agent behavior で観測された 5 つの典型 pattern も追加

これらの変更は、明確な instruction があるにもかかわらず agent が skill 使用を rationalize して回避する挙動に対処するものです。強い wording と先回りした反論により、非準拠を難しくすることを狙っています。

### Files Changed
- Updated: `skills/using-superpowers/SKILL.md` - skill-skipping の rationalization を防ぐ 3 層の enforcement を追加

## v3.2.1 (2025-10-20)

### New Features

**code reviewer agent が plugin に含まれるように**
- `superpowers:code-reviewer` agent を plugin の `agents/` directory に追加
- agent は、plan と coding standard に照らした体系的な code review を提供
- 以前は user 側の personal agent configuration が必要でした
- すべての skill 参照を namespace 付きの `superpowers:code-reviewer` に更新
- Fixes #55

### Files Changed
- New: `agents/code-reviewer.md` - review checklist と output format を持つ agent 定義
- Updated: `skills/requesting-code-review/SKILL.md` - `superpowers:code-reviewer` への参照
- Updated: `skills/subagent-driven-development/SKILL.md` - `superpowers:code-reviewer` への参照

## v3.2.0 (2025-10-18)

### New Features

**brainstorming workflow に design documentation を追加**
- brainstorming skill に Phase 4: Design Documentation を追加
- implementation 前に、design document を `docs/plans/YYYY-MM-DD-<topic>-design.md` へ書き出すように変更
- skill 化の際に失われていた、元の brainstorming command の機能を復元
- document は worktree setup と implementation planning の前に書き出されます
- 時間的なプレッシャー下でも遵守されることを subagent で test 済み

### Breaking Changes

**skill 参照 namespace を標準化**
- すべての内部 skill 参照に `superpowers:` namespace prefix を付けるよう変更
- 形式を `superpowers:test-driven-development` のように更新（以前は `test-driven-development` のみ）
- 影響範囲は、すべての REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL、REQUIRED BACKGROUND 参照
- Skill tool での呼び出し方法と整合します
- 更新した file: brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### Improvements

**design と implementation plan の命名を整理**
- design document は filename collision を防ぐため `-design.md` suffix を使用
- implementation plan は既存の `YYYY-MM-DD-<feature-name>.md` format を継続
- どちらも `docs/plans/` directory に保存し、命名で明確に区別

## v3.1.1 (2025-10-17)

### Bug Fixes

- **README の command syntax を修正** (#44) - すべての command 参照を正しい namespace 付き syntax（`/superpowers:brainstorm` など。`/brainstorm` ではない）に更新しました。plugin 提供 command は、plugin 間の衝突を避けるため Claude Code により自動的に namespace されます。

## v3.1.0 (2025-10-17)

### Breaking Changes

**skill 名を lowercase に標準化**
- すべての skill frontmatter の `name:` field を、directory 名に一致する lowercase kebab-case に統一
- 例: `brainstorming`、`test-driven-development`、`using-git-worktrees`
- skill の告知や cross-reference もすべて lowercase 形式に更新
- directory 名、frontmatter、documentation 全体で一貫した naming を保証します

### New Features

**brainstorming skill を強化**
- phase、activity、tool usage を示す Quick Reference table を追加
- 進捗追跡用に copy できる workflow checklist を追加
- 以前の phase に戻るべきかを判断する decision flowchart を追加
- AskUserQuestion tool の包括的 guidance と具体例を追加
- 構造化質問と open-ended 質問の使い分けを説明する "Question Patterns" section を追加
- Key Principles を一覧しやすい table に再構成

**Anthropic best practices integration**
- `skills/writing-skills/anthropic-best-practices.md` を追加 - Anthropic 公式の skill authoring guide
- 包括的 guidance として writing-skills SKILL.md から参照
- progressive disclosure、workflow、evaluation の pattern を提供

### Improvements

**skill cross-reference の明確化**
- すべての skill 参照に明示的 requirement marker を使用するよう変更:
  - `**REQUIRED BACKGROUND:**` - 理解しておくべき前提
  - `**REQUIRED SUB-SKILL:**` - workflow 内で必ず使う skill
  - `**Complementary skills:**` - 任意だが役立つ関連 skill
- 古い path 形式（`skills/collaboration/X` → `X`）を削除
- Integration section を、分類済みの関係（Required と Complementary）に更新
- cross-reference documentation を best practices に合わせて更新

**Anthropic best practices との整合**
- description の grammar と voice を修正（完全に third-person 化）
- 一覧しやすい Quick Reference table を追加
- Claude が copy して追跡できる workflow checklist を追加
- 自明でない decision point に flowchart を適切に使用
- table format の視認性を改善
- すべての skill を 500 行未満という推奨内に収める

### Bug Fixes

- **不足していた command redirect を再追加** - v3.0 移行時に誤って削除されていた `commands/brainstorm.md` と `commands/write-plan.md` を復元
- `defense-in-depth` の名前不一致を修正（以前は `Defense-in-Depth-Validation`）
- `receiving-code-review` の名前不一致を修正（以前は `Code-Review-Reception`）
- `commands/brainstorm.md` の参照を正しい skill 名に修正
- 存在しない related skills への参照を削除

### Documentation

**writing-skills の改善**
- cross-referencing guidance を明示的 requirement marker に合わせて更新
- Anthropic 公式 best practices への参照を追加
- 適切な skill reference format を示す example を改善

## v3.0.1 (2025-10-16)

### Changes

Anthropic の first-party skills system を使うようになりました！

## v2.0.2 (2025-10-12)

### Bug Fixes

- **local skills repo が upstream より先行しているときの誤警告を修正** - 初期化 script が、local repository に upstream より先の commit がある場合でも誤って "New skills available from upstream" と警告していました。ロジックを修正し、3 つの git state（local が behind = 更新すべき、local が ahead = 警告不要、diverged = 警告すべき）を正しく区別するようにしました。

## v2.0.1 (2025-10-12)

### Bug Fixes

- **plugin context での session-start hook 実行を修正** (#8, PR #9) - hook が "Plugin hook error" で無言失敗し、skills context が読み込まれない問題を修正しました。修正内容:
  - Claude Code の実行 context で BASH_SOURCE が未束縛な場合に備え、`${BASH_SOURCE[0]:-$0}` fallback を使用
  - status flag を絞り込む際に grep 結果が空でも問題にならないよう、`|| true` を追加

---

# Superpowers v2.0.0 リリースノート

## 概要

Superpowers v2.0 は、大きな architecture 変更により、skills をより使いやすく、保守しやすく、community 主導にします。

最大の変更点は **skills repository の分離** です。すべての skills、scripts、documentation が plugin から専用 repository（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）へ移されました。これにより superpowers は、monolithic な plugin から、skills repository の local clone を管理する lightweight shim へと変わります。skills は session start 時に自動更新されます。ユーザーは標準的な git workflow を通じて fork し、改善を contribute できます。skills library は plugin とは独立して version 管理されます。

infrastructure 以外でも、この release では problem-solving、research、architecture に焦点を当てた 9 個の新しい skill を追加しました。中核となる **using-skills** documentation も、命令形の tone とより明確な構成で全面的に書き直し、Claude がいつ・どのように skill を使うべきか理解しやすくしました。**find-skills** は Read tool にそのまま貼り付けられる path を出力するようになり、skill discovery workflow の摩擦を減らしています。

ユーザー体験はシームレスです: plugin が clone、fork、update を自動で処理します。contributor にとっては、この新 architecture により skill の改善と共有が非常に簡単になります。この release は、community resource として skills が素早く進化していくための基盤を築くものです。

## Breaking Changes

### Skills Repository Separation

**最大の変更:** skills はもう plugin 内に存在しません。[obra/superpowers-skills](https://github.com/obra/superpowers-skills) という別 repository へ移されました。

**これが意味すること:**

- **初回 install:** plugin が自動的に skills を `~/.config/superpowers/skills/` へ clone
- **fork:** setup 中に skills repo を fork する option が提示されます（`gh` が install 済みの場合）
- **更新:** skills は session start 時に自動更新されます（可能なら fast-forward）
- **contributing:** branch 上で作業し、local に commit して、upstream へ PR を送る workflow になります
- **shadowing 廃止:** 旧 2 層 system（personal/core）は単一 repo の branch workflow に置き換えられました

**Migration:**

既存 install がある場合:
1. 古い `~/.config/superpowers/.git` は `~/.config/superpowers/.git.bak` に backup されます
2. 古い skills は `~/.config/superpowers/skills.bak` に backup されます
3. `~/.config/superpowers/skills/` に obra/superpowers-skills の fresh clone が作成されます

### Removed Features

- **Personal superpowers overlay system** - git branch workflow に置き換え
- **setup-personal-superpowers hook** - initialize-skills.sh に置き換え

## New Features

### Skills Repository Infrastructure

**Automatic Clone & Setup** (`lib/initialize-skills.sh`)
- 初回実行時に obra/superpowers-skills を clone
- GitHub CLI が install されていれば fork 作成を提案
- upstream/origin remote を正しく設定
- 旧 install からの migration に対応

**Auto-Update**
- 毎回の session start で tracking remote から fetch
- 可能なら fast-forward で自動 merge
- 手動 sync が必要な場合に通知（branch が diverged）
- 手動 sync には pulling-updates-from-skills-repository skill を使用

### New Skills

**Problem-Solving Skills** (`skills/problem-solving/`)
- **collision-zone-thinking** - 無関係な概念を強制的にぶつけ、創発的な insight を引き出す
- **inversion-exercise** - 前提を反転させて隠れた制約をあぶり出す
- **meta-pattern-recognition** - domain をまたぐ普遍的原則を見つける
- **scale-game** - 極端な条件で試し、根本的な真実を露出させる
- **simplification-cascades** - 複数 component を不要にできる insight を見つける
- **when-stuck** - 適切な problem-solving technique へ dispatch する

**Research Skills** (`skills/research/`)
- **tracing-knowledge-lineages** - idea が時間とともにどう進化したかを理解する

**Architecture Skills** (`skills/architecture/`)
- **preserving-productive-tensions** - 早まって 1 つに決めつけず、複数の妥当な approach を保つ

### Skills Improvements

**using-skills（旧 getting-started）**
- getting-started から using-skills に改名
- 命令形 tone で全面書き直し（v4.0.0）
- 重要ルールを先頭に配置
- すべての workflow に "Why" explanation を追加
- 参照には常に `/SKILL.md` suffix を付与
- 厳格な rule と柔軟な pattern の違いを明確化

**writing-skills**
- cross-referencing guidance を using-skills から移動
- token efficiency section（word count target）を追加
- CSO (Claude Search Optimization) guidance を改善

**sharing-skills**
- 新しい branch-and-PR workflow 向けに更新（v2.0.0）
- personal/core split への参照を削除

**pulling-updates-from-skills-repository**（新規）
- upstream と同期する完全な workflow を提供
- 旧 "updating-skills" skill を置き換え

### Tools Improvements

**find-skills**
- `/SKILL.md` suffix 付きの完全 path を出力するように変更
- Read tool から直接使える path になります
- help text を更新

**skill-run**
- scripts/ から skills/using-skills/ へ移動
- documentation を改善

### Plugin Infrastructure

**Session Start Hook**
- skills repository の location から読み込むように変更
- session start 時に完全な skills list を表示
- skills location 情報を出力
- update status（updated successfully / behind upstream）を表示
- "skills behind" warning を出力末尾へ移動

**Environment Variables**
- `SUPERPOWERS_SKILLS_ROOT` を `~/.config/superpowers/skills` に設定
- すべての path で一貫して使用

## Bug Fixes

- fork 時に upstream remote を重複追加していた問題を修正
- find-skills の出力で `skills/` prefix が二重になる問題を修正
- session-start から obsolete な setup-personal-superpowers 呼び出しを削除
- hooks と commands 全体の path 参照を修正

## Documentation

### README
- 新しい skills repository architecture に合わせて更新
- superpowers-skills repo への目立つ link を追加
- auto-update の説明を更新
- skill 名と参照を修正
- Meta skills list を更新

### Testing Documentation
- 包括的な testing checklist（`docs/TESTING-CHECKLIST.md`）を追加
- testing 用の local marketplace config を作成
- 手動 testing scenario を文書化

## Technical Details

### File Changes

**追加:**
- `lib/initialize-skills.sh` - skills repo の初期化と auto-update
- `docs/TESTING-CHECKLIST.md` - 手動 testing scenario
- `.claude-plugin/marketplace.json` - local testing config

**削除:**
- `skills/` directory（82 files）- 以後は obra/superpowers-skills に配置
- `scripts/` directory - 以後は obra/superpowers-skills/skills/using-skills/ に配置
- `hooks/setup-personal-superpowers.sh` - obsolete

**変更:**
- `hooks/session-start.sh` - `~/.config/superpowers/skills` から skills を使用
- `commands/brainstorm.md` - path を SUPERPOWERS_SKILLS_ROOT に更新
- `commands/write-plan.md` - path を SUPERPOWERS_SKILLS_ROOT に更新
- `commands/execute-plan.md` - path を SUPERPOWERS_SKILLS_ROOT に更新
- `README.md` - 新 architecture に合わせて全面書き直し

### Commit History

この release には以下が含まれます:
- skills repository 分離のための 20 以上の commit
- PR #1: Amplifier に着想を得た problem-solving / research skills
- PR #2: Personal superpowers overlay system（後に置き換え）
- 複数の skill 改良と documentation 改善

## Upgrade Instructions

### Fresh Install

```bash
# In Claude Code
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

plugin がすべて自動で処理します。

### v1.x からのアップグレード

1. **personal skills を backup**（ある場合）:
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **plugin を更新:**
   ```bash
   /plugin update superpowers
   ```

3. **次回 session start 時:**
   - 古い install は自動的に backup されます
   - fresh な skills repo が clone されます
   - GitHub CLI があれば fork の option が提示されます

4. **personal skills を移行**（ある場合）:
   - local skills repo に branch を作成
   - backup から personal skills をコピー
   - commit して fork へ push
   - 可能なら PR で upstream へ還元を検討

## What's Next

### For Users

- 新しい problem-solving skills を試す
- skill 改善のための branch-based workflow を試す
- skills を community に還元する

### For Contributors

- skills repository は現在 https://github.com/obra/superpowers-skills にあります
- Fork → Branch → PR workflow
- documentation への TDD approach については skills/meta/writing-skills/SKILL.md を参照

## Known Issues

現時点で特になし。

## Credits

- Amplifier pattern に着想を得た problem-solving skills
- community からの contribution と feedback
- skill 効果に対する広範な testing と改善

---

**Full Changelog:** https://github.com/obra/superpowers/compare/dd013f6...main
**Skills Repository:** https://github.com/obra/superpowers-skills
**Issues:** https://github.com/obra/superpowers/issues
