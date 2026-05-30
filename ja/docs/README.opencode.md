# OpenCode 向け Superpowers

[OpenCode.ai](https://opencode.ai) で Superpowers を使うための完全ガイドです。

## インストール

グローバルまたはプロジェクトレベルの `opencode.json` の `plugin` 配列に superpowers を追加します:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

OpenCode を再起動してください。プラグインは OpenCode のプラグインマネージャー経由でインストールされ、すべてのスキルが登録されます。

確認するには、次のように尋ねます: "Tell me about your superpowers"

OpenCode は独自のプラグインインストール機構を使います。Claude Code、Codex、または別の harness も使っている場合は、それぞれに対して個別に Superpowers をインストールしてください。

### 古いシンボリックリンクベースのインストールからの移行

以前に `git clone` とシンボリックリンクで superpowers をインストールしていた場合は、古い設定を削除してください:

```bash
# 古いシンボリックリンクを削除
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 必要に応じてクローンしたリポジトリも削除
rm -rf ~/.config/opencode/superpowers

# superpowers 用に追加した skills.paths があれば opencode.json から削除
```

その後、上記のインストール手順に従ってください。

## 使い方

### スキルを探す

利用可能なスキルを一覧表示するには、OpenCode のネイティブな `skill` ツールを使います:

```
use skill tool to list skills
```

### スキルを読み込む

```
use skill tool to load superpowers/brainstorming
```

### 個人用スキル

自分用のスキルは `~/.config/opencode/skills/` に作成できます:

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

`~/.config/opencode/skills/my-skill/SKILL.md` を作成します:

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### プロジェクト用スキル

プロジェクト固有のスキルは、プロジェクト内の `.opencode/skills/` に作成します。

**スキルの優先順位:** プロジェクト用スキル > 個人用スキル > Superpowers スキル

## 更新

OpenCode は git ベースのパッケージ指定を通じて Superpowers をインストールします。一部の OpenCode および Bun のバージョンでは、解決済みの git 依存関係が lockfile やキャッシュに固定されるため、再起動しても最新の Superpowers コミットが反映されないことがあります。更新が反映されない場合は、OpenCode のパッケージキャッシュをクリアするか、プラグインを再インストールしてください。

特定のバージョンに固定するには、ブランチまたはタグを使います:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 仕組み

このプラグインは次の 2 つを行います:

1. `experimental.chat.system.transform` フックを通じて**ブートストラップコンテキストを注入**し、すべての会話に superpowers の認識を追加します。
2. `config` フックを通じて**skills ディレクトリを登録**し、シンボリックリンクや手動設定なしで OpenCode がすべての superpowers スキルを検出できるようにします。

### ツール対応

Claude Code 向けに書かれたスキルは、自動的に OpenCode 向けへ適応されます:

- `TodoWrite` → `todowrite`
- サブエージェント付きの `Task` → OpenCode の `@mention` システム
- `Skill` ツール → OpenCode ネイティブの `skill` ツール
- ファイル操作 → OpenCode ネイティブツール

## トラブルシューティング

### プラグインが読み込まれない

1. OpenCode のログを確認します: `opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. `opencode.json` のプラグイン行が正しいことを確認します
3. OpenCode のバージョンが十分新しいことを確認します

### Windows でのインストール問題

一部の Windows 版 OpenCode では、`git+https` URL のキャッシュパスや、通常のターミナルでは動いていても Bun が `git.exe` を見つけられない問題など、git ベースのプラグイン指定に関する上流のインストーラー問題があります。OpenCode がプラグインをインストールできない場合は、システムの npm でインストールし、OpenCode からローカルパッケージを指すようにしてみてください:

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

次に、`opencode.json` でインストール済みパッケージのパスを使います:

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### スキルが見つからない

1. OpenCode の `skill` ツールで利用可能なスキルを一覧表示します
2. プラグインが読み込まれているか確認します（上記参照）
3. 各スキルには、有効な YAML frontmatter を持つ `SKILL.md` ファイルが必要です

### ブートストラップが表示されない

1. OpenCode のバージョンが `experimental.chat.system.transform` フックをサポートしていることを確認します
2. 設定変更後に OpenCode を再起動します

## サポート

- 問題の報告: https://github.com/obra/superpowers/issues
- メインドキュメント: https://github.com/obra/superpowers
- OpenCode ドキュメント: https://opencode.ai/docs/
