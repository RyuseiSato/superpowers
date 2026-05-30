# Worktree Rototill: Detect-and-Defer

**Date:** 2026-04-06
**Status:** Draft
**Ticket:** PRI-974
**Subsumes:** PRI-823（Codex App compatibility）

## 問題

Superpowers は worktree 管理について強い前提を持っています。具体的には、特定のパス（`.worktrees/<branch>`）、特定の command（`git worktree add`）、特定の cleanup（`git worktree remove`）です。一方で、Claude Code、Codex App、Gemini CLI、Cursor はいずれも独自のパス、ライフサイクル管理、cleanup を持つ native な worktree support を提供します。

このため、3 つの failure mode が生じます。

1. **重複** — Claude Code では、この skill が `EnterWorktree`/`ExitWorktree` の既存機能を重複実装してしまう
2. **衝突** — Codex App では、skill がすでに管理されている worktree の中でさらに worktree を作ろうとする
3. **Phantom state** — `.worktrees/` に skill が作った worktree は harness から見えず、`.claude/worktrees/` に harness が作った worktree は skill から見えない

native support を持たない harness（Codex CLI、OpenCode、Copilot standalone）では、superpowers は実際のギャップを埋めています。この skill はなくすべきではなく、native support が存在する時には前に出ないようにすべきです。

## 目標

1. native な harness の worktree system がある場合はそれを優先する
2. それがない harness には引き続き worktree support を提供する
3. `finishing-a-development-branch` の既知の 3 バグを修正する（#940、#999、#238）
4. worktree 作成を必須ではなく opt-in にする（#991）
5. ハードコードされた `CLAUDE.md` 参照を platform-neutral な言い回しに置き換える（#1049）

## 非目標

- worktree ごとの環境慣習（`.worktree-env.sh`, port offsetting） — Phase 4
- path enforcement のための PreToolUse hooks — Phase 4
- multi-repo worktree のドキュメント — Phase 4
- worktree に関する brainstorming checklist の変更 — Phase 4
- `.superpowers-session.json` の metadata tracking（興味深い PR #997 のアイデアだが、v1 には不要）
- worktree への hooks symlink（PR #965 のアイデア、別の関心事）

## 設計原則

### platform ではなく状態を検出する

どの harness かを環境変数で嗅ぎ分けるのではなく、`GIT_DIR != GIT_COMMON` を使って「私はすでに worktree の中にいるか？」を判定します。これは安定した git のプリミティブ（git 2.5 以降、2015 年から）で、すべての harness で普遍的に機能し、新しい harness が現れてもメンテナンスは不要です。

### 宣言的な意図、規定された fallback

skill は目標（「作業が isolated workspace で行われることを保証する」）を記述し、native tools がある場合はそれに委ねます。具体的な git command を規定するのは、native worktree support を持たない harness 向けの fallback としてだけです。Step 1a が先に来て native tools を明示的に挙げ（`EnterWorktree`, `WorktreeCreate`, `/worktree`, `--worktree`）、Step 1b で git fallback を示します。元の spec では Step 1a は抽象的（"you know your own toolkit"）でしたが、TDD により、Step 1a が曖昧すぎると agent が Step 1b の具体的な command に引き寄せられることが分かりました。信頼できる優先付けにするには、明示的な tool naming と consent-authorization bridge が必要でした。

### provenance ベースの所有権

worktree を作成した主体が、その cleanup の責任を持ちます。harness が作ったなら、superpowers は触れません。superpowers が作ったなら（git fallback 経由）、superpowers が cleanup します。ヒューリスティックはこうです。worktree が `.worktrees/` または `~/.config/superpowers/worktrees/` 配下にあるなら superpowers の所有物。それ以外（`.claude/worktrees/`, `~/.codex/worktrees/`, `.gemini/worktrees/`）は harness の所有物です。

## 設計

### 1. `using-git-worktrees` SKILL.md の書き換え

この skill は、作成前に 3 つの新しい step を持つようになり、作成フローは単純化されます。

#### Step 0: 既存の分離状態を検出する

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

結果は 3 通りです。

| Condition | Meaning | Action |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 通常の repo checkout | Step 0.5 へ進む |
| `GIT_DIR != GIT_COMMON`, named branch | すでに linked worktree にいる | Step 3（project setup）へスキップ。報告: "Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`, detached HEAD | 外部管理の worktree（例: Codex App sandbox） | Step 3 へスキップ。報告: "Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

Step 0 は、誰が worktree を作ったかや、どの harness が動いているかを気にしません。由来が何であれ、worktree は worktree です。

**Submodule guard:** `GIT_DIR != GIT_COMMON` は git submodule の中でも true になります。「すでに worktree にいる」と結論づける前に、submodule ではないことを確認します。

```bash
# これがパスを返したら、worktree ではなく submodule にいる
git rev-parse --show-superproject-working-tree 2>/dev/null
```

submodule 内なら `GIT_DIR == GIT_COMMON` と同様に扱い（Step 0.5 へ進む）、worktree 扱いはしません。

#### Step 0.5: 同意

Step 0 で既存の分離状態が見つからなかった場合（`GIT_DIR == GIT_COMMON`）、作成前に確認します。

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

yes なら Step 1 へ進みます。no ならその場で作業し、worktree は作らず Step 3 へ進みます。

この step は、Step 0 が既存の分離状態を検出した場合には完全にスキップされます（すでに存在するものについて尋ねる意味はないため）。

#### Step 1a: Native Tools（優先）

> The user has asked for an isolated workspace (Step 0 consent). Check your available tools — do you have `EnterWorktree`, `WorktreeCreate`, a `/worktree` command, or a `--worktree` flag? If YES: the user's consent to create a worktree is your authorization to use it. Use it now and skip to Step 3.

native tool を使ったら、Step 3（project setup）へ進みます。

**Design note — TDD revision:** 元の spec では、Step 1a は意図的に短く抽象的でした（"You know your own toolkit — the skill does not need to name specific tools"）。TDD 検証はこれを否定しました。agent は Step 1b の具体的な git command に引き寄せられ、抽象的なガイダンスを無視したのです（合格率 2/6）。次の 3 つの変更で解決しました（GREEN と PRESSURE テスト合計で 50/50 の合格率）。

1. **明示的な tool naming** — `EnterWorktree`, `WorktreeCreate`, `/worktree`, `--worktree` を名前で列挙することで、判断が解釈（"native tool があるか？"）から事実確認（"自分の tool list に `EnterWorktree` はあるか？"）へ変わります。これらの tool を持たない platform の agent は単に確認して見つからず、Step 1b へ自然に落ちます。false positive は観測されませんでした。
2. **Consent bridge** — "the user's consent to create a worktree is your authorization to use it" という文言で、`EnterWorktree` の tool-level guardrail（"ONLY when user explicitly asks"）を直接扱います。tool description は skill instructions より優先されるため（Claude Code #29950）、skill 側で user consent を tool が必要とする authorization として位置づける必要があります。
3. **Red Flag entry** — Red Flags section で具体的なアンチパターンを名指しすること（"Use `git worktree add` when you have a native worktree tool — this is the #1 mistake"）。

ファイル分割（Step 1b を別 skill に切り出す）は検証され、不要だと分かりました。アンカリング問題は git command を物理的に分離することではなく、Step 1a の文面の質で解決されます。full 240-line skill（すべての git command が見える状態）での control test でも 20/20 で通過しました。

#### Step 1b: Git Worktree Fallback

native tool が利用できない場合は、手動で worktree を作成します。

**Directory selection**（優先順）:
1. 既存の `.worktrees/` または `worktrees/` directory を確認し、見つかればそれを使う。両方ある場合は `.worktrees/` を優先する。
2. 既存の `~/.config/superpowers/worktrees/<project>/` directory を確認し、見つかればそれを使う（旧 global path との後方互換性）。
3. プロジェクトの agent instruction file（CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, または同等のもの）に、worktree directory の好みが書かれていないか確認する。
4. 既定値は `.worktrees/`。

対話式の directory selection prompt はありません。global path（`~/.config/superpowers/worktrees/`）は新規ユーザー向けの選択肢としては提示しませんが、そこに既存 worktree がある場合は後方互換のため検出して利用します。

**Safety verification**（project-local directories のみ）:

```bash
git check-ignore -q .worktrees 2>/dev/null
```

ignore されていなければ、先に `.gitignore` に追加して commit してから進みます。

**作成:**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**Hooks awareness:** Git worktree は親 repo の hooks directory を継承しません。1b で worktree を作成した後、main repo に hooks directory が存在するならそれを symlink します。

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

これにより、作業が worktree に移った途端に pre-commit checks、linters、その他の hooks が黙って止まるのを防げます。（PR #965 のアイデア。）

**Sandbox fallback:** `git worktree add` が permission error で失敗した場合は、制限付き環境とみなします。作成をスキップし、現在の directory で作業して Step 3 へ進みます。

**Step numbering note:** 現行 skill では Steps 1-4 がフラットなリストです。この再設計では 0、0.5、1a、1b、3、4 を使います。Step 2 はありません — 旧来の単一の "Create Isolated Workspace" が 1a/1b 構造に分割されたためです。実装では、きれいに番号を振り直す（例: 0 → "Step 0: Detect", 0.5 → Step 0 の内部フロー、1a/1b → "Step 1", 3 → "Step 2", 4 → "Step 3"）か、現行番号を維持して注釈を付けるかのどちらでも構いません。implementer の判断に委ねます。

#### Steps 3-4: Project Setup と Baseline Tests（変更なし）

どの経路で workspace が作られたかに関係なく（Step 0 で既存検出、Step 1a の native tool、Step 1b の git fallback、あるいは worktree を作らない場合）、実行はここに収束します。

- **Step 3:** project setup を自動検出して実行する（`npm install`, `cargo build`, `pip install`, `go mod download` など）
- **Step 4:** テストスイートを実行する。テストが失敗したら、失敗内容を報告し、続行するか尋ねる

### 2. `finishing-a-development-branch` SKILL.md の書き換え

finishing skill は環境検出を取り込み、3 つのバグを修正します。

#### Step 1: テストを確認する（変更なし）

プロジェクトのテストスイートを実行します。失敗したら止めます。完了 options は提示しません。

#### Step 1.5: 環境を検出する（新規）

作成側の Step 0 と同じ検出を再実行します。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

3 つの経路があります。

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（通常 repo） | 標準の 4 options | cleanup すべき worktree はない |
| `GIT_DIR != GIT_COMMON`, named branch | 標準の 4 options | provenance ベース（Step 5 を参照） |
| `GIT_DIR != GIT_COMMON`, detached HEAD | 簡略 menu: 新規 branch として push + PR、現状維持、破棄 | merge option なし（detached HEAD からは merge できない） |

#### Step 2: Base Branch を決定する（変更なし）

#### Step 3: Options を提示する

**通常 repo と named-branch worktree:**

1. `<base-branch>` にローカルで merge する
2. Push して Pull Request を作成する
3. branch をそのまま保持する（あとで自分で対応する）
4. この作業を破棄する

**Detached HEAD:**

1. 新しい branch として push して Pull Request を作成する
2. そのまま保持する（あとで自分で対応する）
3. この作業を破棄する

#### Step 4: 選択を実行する

**Option 1（ローカルで merge）:**

```bash
# CWD safety のため main repo root を取得（Bug #238 fix）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# まず merge し、何かを消す前に成功を確認する
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# merge 成功後にのみ: worktree を削除し、その後 branch を削除する（Bug #999 fix）
git worktree remove "$WORKTREE_PATH"  # superpowers が所有する場合のみ
git branch -d <feature-branch>
```

順序が重要です: merge → 確認 → worktree 削除 → branch 削除。旧 skill は worktree を削除する前に branch を削除していました（worktree がまだその branch を参照しているため失敗します）。逆に、素朴に先に worktree を削除するのも誤りです。もしその後 merge が失敗すると、作業ディレクトリが消え、変更も失われるからです。

**Option 2（PR を作成）:**

branch を push して PR を作成します。worktree は cleanup **しません** — ユーザーが PR の反復作業に必要だからです。（Bug #940 fix: 矛盾する "Then: Cleanup worktree" という prose を削除。）

**Option 3（そのまま保持）:** アクションなし。

**Option 4（破棄）:** `discard` とタイプする確認を必須にします。その後、worktree を削除し（superpowers が所有する場合のみ）、branch を force-delete します。

#### Step 5: Cleanup（更新）

```
if GIT_DIR == GIT_COMMON:
    # 通常 repo。cleanup すべき worktree はない
    done

if worktree path is under .worktrees/ or ~/.config/superpowers/worktrees/:
    # Superpowers が作成したもの — cleanup は自分たちの責任
    cd to main repo root       # Bug #238 fix
    git worktree remove <path>

else:
    # Harness が作成したもの — 引き渡す
    # platform が workspace-exit tool を提供するならそれを使う
    # そうでなければ worktree はそのまま残す
```

cleanup を行うのは Options 1 と 4 のみです。Options 2 と 3 は常に worktree を保持します。（Bug #940 fix。）

**Stale worktree pruning:** `git worktree remove` を実行した後は必ず `git worktree prune` も走らせ、self-healing step とします。worktree directory は harness cleanup、手動の `rm`、あるいは `.claude/` cleanup などにより帯域外で削除されることがあり、その結果 stale registration が残って紛らわしいエラーを生みます。1 行で silent rot を防げます。（PR #1072 からのアイデア。）

### 3. Integration Updates

#### `subagent-driven-development` と `executing-plans`

両方とも Integration sections に `using-git-worktrees` を REQUIRED として記載しています。これを次に変更します。

> `using-git-worktrees` — Ensures isolated workspace (creates one or verifies existing)

skill 自身が consent（Step 0.5）と detection（Step 0）を扱うため、呼び出し側 skill が gate や prompt を持つ必要はありません。

#### `writing-plans`

"should be run in a dedicated worktree (created by brainstorming skill)" という古い主張を削除します。brainstorming は design skill であり、worktree は作成しません。worktree prompt は execution 時に `using-git-worktrees` 経由で行われます。

### 4. Platform-Neutral Instruction File References

worktree 関連 skill にあるハードコードされた `CLAUDE.md` は、すべて次の表現に置き換えます。

> "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"

これは Step 1b の directory preference checks のすべてに適用されます。

## バグ修正（同梱）

| Bug | Problem | Fix | Location |
|-----|---------|-----|----------|
| #940 | Option 2 の prose に "Then: Cleanup worktree (Step 5)" とあるが、quick reference では保持すると言っている。Step 5 は "For Options 1, 2, 4" と言う一方、Common Mistakes では "Options 1 and 4 only" と言っている。 | Option 2 から cleanup を削除。Step 5 は Options 1 と 4 のみに適用。 | finishing SKILL.md |
| #999 | Option 1 が worktree を削除する前に branch を削除している。`git branch -d` は、worktree がまだその branch を参照しているため失敗しうる。 | 順序を merge → verify tests → remove worktree → delete branch に変更。何かを削除する前に merge が成功していなければならない。 | finishing SKILL.md |
| #238 | 削除対象の worktree 内に CWD があると、`git worktree remove` が黙って失敗する。 | CWD guard を追加: `git worktree remove` の前に main repo root へ `cd` する。 | finishing SKILL.md |

## 解決される Issue

| Issue | Resolution |
|-------|-----------|
| #940 | 直接修正（Bug #940） |
| #991 | Step 0.5 の opt-in consent |
| #918 | Step 0 の検出 + Step 1.5 の finishing 時検出 |
| #1009 | Step 1a により解決 — agent は native tools（例: `EnterWorktree`）を使い、harness-native な path に作成する。Step 1a が機能することが前提。Risks を参照。 |
| #999 | 直接修正（Bug #999） |
| #238 | 直接修正（Bug #238） |
| #1049 | platform-neutral な instruction file 参照 |
| #279 | detect-and-defer で解決 — native path を上書きしないため尊重される |
| #574 | **Deferred.** この spec は、バグのある brainstorming skill には触れない。完全な修正（brainstorming の checklist に worktree step を追加）は Phase 4。 |

## リスク

### Step 1a は荷重を支える前提 — 解決済み

Step 1a、すなわち agent が git fallback より native worktree tool を優先するという前提が、この設計全体の基盤です。native support のある harness で agent が Step 1a を無視して Step 1b に落ちると、detect-and-defer は完全に失敗します。

**Status:** このリスクは実装中に実際に顕在化しました。元の抽象的な Step 1a（"You know your own toolkit"）は Claude Code で 2/6 しか通りませんでした。TDD gate は設計どおり機能し、skill files が変更される前にこの失敗を検出して、壊れたリリースを防ぎました。3 回の REFACTOR iteration により根本原因（具体的な command への agent anchoring、skill instructions より優先される tool-description guardrail）が特定され、GREEN と PRESSURE テストで 50/50 検証された修正が得られました。詳細は上記の Step 1a design note を参照してください。

**Cross-platform validation:**

2026-04-06 時点で、agent-callable な mid-session worktree tool（`EnterWorktree`）を持つ harness は Claude Code だけです。その他は、agent 開始前に worktree を作る（Codex App、Gemini CLI、Cursor）か、native worktree support がありません（Codex CLI、OpenCode）。Step 1a は forward-compatible です。他の harness が agent-callable な worktree tool を追加した際も、agent は列挙された例と照合して、それを skill 変更なしで使えるようになります。

| Harness | 現在の worktree model | Skill mechanism | Tested |
|---------|----------------------|-----------------|--------|
| Claude Code | Agent-callable `EnterWorktree` | Step 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | native tool なし（shell のみ） | Step 1b git fallback | 6/6（`codex exec`） |
| Gemini CLI | 起動時 `--worktree` flag、agent tool なし | flag 付きで起動された場合は Step 0、そうでなければ Step 1b | Step 0: 1/1, Step 1b: 1/1（`gemini -p`） |
| Cursor Agent | ユーザー向け `/worktree`、agent tool なし | ユーザーが有効化済みなら Step 0、そうでなければ Step 1b | Step 0: 1/1, Step 1b: 1/1（`cursor-agent -p`） |
| Codex App | platform 管理、detached HEAD、agent tool なし | Step 0 で既存状態を検出 | 1/1 simulated |
| OpenCode | 検出のみ（`ctx.worktree`）、agent tool なし | Step 1b git fallback | Untested（CLI access なし） |

**Residual risks:**
1. Anthropic が `EnterWorktree` の tool description をより制限的に変更した場合（例: "Do not use based on skill instructions"）、consent bridge は壊れます。tool description が skill-driven invocation を許容するよう issue を出す価値があります。
2. 今後ほかの harness が agent-callable な worktree tool を追加した際、それらの名前が Step 1a の list にない可能性があります。新しい tool が現れたら list を更新すべきです。一般化した表現（"a worktree or workspace-isolation tool"）が多少の将来対応にはなっています。

### provenance heuristic

`.worktrees/` または `~/.config/superpowers/worktrees/` = 自分たちのもの、それ以外は hands off、というヒューリスティックは現在のすべての harness で機能します。将来の harness が `.worktrees/` を慣習として採用した場合、false positive になります（superpowers が harness-owned な worktree の cleanup を試みてしまう）。同様に、ユーザーが superpowers を使わず手動で `git worktree add .worktrees/experiment` を実行した場合も、誤って所有権を主張します。いずれも低リスクです。各 harness は branded path を使っており、手動で `.worktrees/` を作るケースもまれですが、留意は必要です。

### Detached HEAD finishing

detached HEAD worktree 向けの簡略 menu（merge option なし）は、Codex App の sandbox model には正しい設計です。ユーザーが別の理由で detached HEAD にいる場合でも、この簡略 menu は妥当です。そもそも detached HEAD からは branch を作らない限り merge できません。

## 実装メモ

両 skill file には、実装時に core steps 以外にも更新すべき section があります。

- **Frontmatter**（`name`, `description`）: detect-and-defer の挙動を反映するよう更新
- **Quick Reference tables**: 新しい step structure と bug fixes に合わせて書き換え
- **Common Mistakes sections**: 旧挙動を前提にした項目（例: "Skip CLAUDE.md check" はもはや誤り）を更新または削除
- **Red Flags sections**: 新しい優先順位を反映（例: "Step 0 が既存の isolation を検出したら絶対に worktree を作らない"）
- **Integration sections**: skill 間の相互参照を更新

この spec は *何を変えるか* を説明するものです。secondary sections の正確な編集内容は implementation plan で規定します。

## 今後の作業（この spec の対象外）

- **Phase 3 remainder:** `$TMPDIR` directory option（#666）、caching と env inheritance の setup docs（#299）
- **Phase 4:** path enforcement のための PreToolUse hooks（#1040）、per-worktree env conventions（#597）、brainstorming checklist の worktree step（#574）、multi-repo documentation（#710）
