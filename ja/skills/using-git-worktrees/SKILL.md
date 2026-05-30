---
name: using-git-worktrees
description: 現在の workspace から隔離して進めたい機能開発を始めるとき、または実装計画を実行する前に使う。ネイティブツールまたは git worktree のフォールバックで隔離された workspace の存在を保証する
---

# Using Git Worktrees

## 概要

作業が隔離された workspace で行われるようにします。まずプラットフォームのネイティブな worktree ツールを優先し、それがない場合にだけ手動の git worktree を使います。

**中核原則:** まず既存の隔離状態を検出する。その次にネイティブツールを使う。その後で git にフォールバックする。ハーネスに逆らわない。

**開始時に宣言する:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Step 0: 既存の隔離状態を検出する

**何かを作る前に、すでに隔離された workspace にいるか確認してください。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Submodule ガード:** `GIT_DIR != GIT_COMMON` は git submodule 内でも真になります。「すでに worktree にいる」と判断する前に、submodule ではないことを確認してください。

```bash
# If this returns a path, you're in a submodule, not a worktree — treat as normal repo
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**`GIT_DIR != GIT_COMMON`（かつ submodule ではない）場合:** すでに linked worktree にいます。Step 3（Project Setup）へ進んでください。別の worktree を作ってはいけません。

ブランチ状態付きで報告します:
- ブランチ上: "Already in isolated workspace at `<path>` on branch `<name>`."
- Detached HEAD: "Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**`GIT_DIR == GIT_COMMON`（または submodule 内）場合:** 通常の repo checkout にいます。

指示の中で、ユーザーがすでに worktree の希望を示していますか？ そうでなければ、作成前に同意を求めてください:

> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

既存の明示された希望があれば、質問せずそれに従ってください。ユーザーが同意しなければ、その場で作業し、Step 3 へ進みます。

## Step 1: 隔離された workspace を作成する

**利用できる手段は 2 つあります。次の順番で試してください。**

### 1a. ネイティブな Worktree ツール（推奨）

ユーザーは隔離された workspace を求めています（Step 0 の同意）。すでに worktree を作る方法がありますか？ `EnterWorktree`、`WorktreeCreate`、`/worktree` コマンド、`--worktree` フラグのような名前のツールかもしれません。あるならそれを使い、Step 3 へ進んでください。

ネイティブツールは、ディレクトリ配置、ブランチ作成、クリーンアップを自動で扱います。ネイティブツールがあるのに `git worktree add` を使うと、ハーネスが認識・管理できない phantom state ができます。

Step 1b に進むのは、利用可能なネイティブ worktree ツールがない場合だけです。

### 1b. Git Worktree フォールバック

**これを使うのは Step 1a が適用できない場合だけです**。つまり、利用可能なネイティブ worktree ツールがない場合です。git を使って手動で worktree を作成します。

#### ディレクトリ選択

次の優先順に従ってください。ユーザーの明示的な希望は、観測した filesystem 状態より常に優先されます。

1. **指示内に宣言済みの worktree ディレクトリ設定がないか確認する。** すでに指定されていれば、質問せずそれを使う。

2. **プロジェクトローカルの既存 worktree ディレクトリを確認する:**
   ```bash
   ls -d .worktrees 2>/dev/null     # Preferred (hidden)
   ls -d worktrees 2>/dev/null      # Alternative
   ```
   見つかったらそれを使います。両方あれば `.worktrees` を優先します。

3. **既存のグローバルディレクトリを確認する:**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ls -d ~/.config/superpowers/worktrees/$project 2>/dev/null
   ```
   見つかったらそれを使います（旧来の global path との後方互換のため）。

4. **他に指針がない場合**、プロジェクトルートの `.worktrees/` をデフォルトにします。

#### 安全性の検証（プロジェクトローカルディレクトリのみ）

**worktree を作る前に、そのディレクトリが ignore されていることを必ず確認してください:**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**ignore されていない場合:** .gitignore に追加し、その変更を commit してから続行してください。

**これが重要な理由:** worktree の内容を誤って repository に commit するのを防ぐためです。

グローバルディレクトリ（`~/.config/superpowers/worktrees/`）には検証は不要です。

#### Worktree を作成する

```bash
project=$(basename "$(git rev-parse --show-toplevel)")

# Determine path based on chosen location
# For project-local: path="$LOCATION/$BRANCH_NAME"
# For global: path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**Sandbox フォールバック:** `git worktree add` が permission error（sandbox denial）で失敗したら、sandbox によって worktree 作成がブロックされたことをユーザーへ伝え、代わりに現在のディレクトリで作業することを報告してください。その後、その場でセットアップとベースラインテストを行います。

## Step 3: プロジェクトセットアップ

適切なセットアップを自動検出して実行します:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## Step 4: クリーンなベースラインを確認する

workspace がクリーンに始まることを確認するためテストを実行します:

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**テストが失敗した場合:** 失敗を報告し、続行するか調査するかを確認します。

**テストが通った場合:** 準備完了を報告します。

### 報告

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## クイックリファレンス

| 状況 | 対応 |
|-----------|--------|
| すでに linked worktree にいる | 作成をスキップ（Step 0） |
| submodule 内にいる | 通常の repo として扱う（Step 0 のガード） |
| ネイティブ worktree ツールがある | それを使う（Step 1a） |
| ネイティブツールがない | Git worktree フォールバック（Step 1b） |
| `.worktrees/` がある | それを使う（ignore を確認） |
| `worktrees/` がある | それを使う（ignore を確認） |
| 両方ある | `.worktrees/` を使う |
| どちらもない | 指示ファイルを確認してから `.worktrees/` を使う |
| global path がある | それを使う（後方互換） |
| ディレクトリが ignore されていない | .gitignore に追加して commit |
| 作成時に permission error | sandbox フォールバック、その場で作業 |
| ベースラインでテスト失敗 | 失敗を報告して確認を取る |
| package.json/Cargo.toml がない | 依存インストールをスキップ |

## よくあるミス

### ハーネスに逆らう

- **問題:** プラットフォームがすでに隔離を提供しているのに `git worktree add` を使う
- **修正:** Step 0 で既存の隔離を検出し、Step 1a でネイティブツールを優先する

### 検出を飛ばす

- **問題:** 既存 worktree の中に入れ子の worktree を作る
- **修正:** 何かを作る前に必ず Step 0 を実行する

### ignore 検証を飛ばす

- **問題:** worktree 内容が追跡され、git status を汚染する
- **修正:** プロジェクトローカル worktree を作る前に必ず `git check-ignore` を使う

### ディレクトリ位置を決め打ちする

- **問題:** 一貫性を壊し、プロジェクト規約に反する
- **修正:** 優先順に従う: existing > global legacy > instruction file > default

### テスト失敗のまま進む

- **問題:** 新しいバグと既存問題の区別ができない
- **修正:** 失敗を報告し、明示的な許可を得てから進む

## レッドフラグ

**絶対にしないこと:**
- Step 0 で既存の隔離が検出されたのに worktree を作る
- ネイティブ worktree ツール（例: `EnterWorktree`）があるのに `git worktree add` を使う。これが最大のミスです。あるならそれを使うこと。
- Step 1a を飛ばして Step 1b の git コマンドへ直行する
- ignore 確認なしで worktree を作る（プロジェクトローカル）
- ベースラインテスト確認を飛ばす
- テスト失敗のまま確認なしで進む

**常に行うこと:**
- まず Step 0 の検出を実行する
- git フォールバックよりネイティブツールを優先する
- ディレクトリ優先順に従う: existing > global legacy > instruction file > default
- プロジェクトローカルではディレクトリが ignore されていることを確認する
- プロジェクトセットアップを自動検出して実行する
- クリーンなテストベースラインを確認する

