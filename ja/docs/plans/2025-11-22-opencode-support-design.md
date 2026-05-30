# OpenCode サポート設計

**日付:** 2025-11-22
**著者:** Bot & Jesse
**状態:** 設計完了、実装待ち

## 概要

既存の Codex 実装とコア機能を共有する、ネイティブな OpenCode プラグインアーキテクチャを使って、OpenCode.ai 向けの superpowers 完全対応を追加する。

## 背景

OpenCode.ai は、Claude Code や Codex に似たコーディングエージェントです。superpowers を OpenCode に移植しようとした過去の試み（PR #93、PR #116）では、ファイルコピー方式が使われていました。この設計では別のアプローチを取ります。つまり、彼らの JavaScript/TypeScript プラグインシステムを使ってネイティブな OpenCode プラグインを構築しつつ、Codex 実装とコードを共有します。

### プラットフォーム間の主な違い

- **Claude Code**: ネイティブな Anthropic プラグインシステム + ファイルベースのスキル
- **Codex**: プラグインシステムなし → bootstrap markdown + CLI スクリプト
- **OpenCode**: イベントフックとカスタムツール API を備えた JavaScript/TypeScript プラグイン

### OpenCode のエージェントシステム

- **Primary agents**: Build（デフォルト、フルアクセス）と Plan（制限付き、読み取り専用）
- **Subagents**: General（調査、検索、マルチステップタスク）
- **Invocation**: Primary agents による自動ディスパッチ、または手動の `@mention` 構文
- **Configuration**: `opencode.json` または `~/.config/opencode/agent/` 内のカスタムエージェント

## アーキテクチャ

### 高レベル構成

1. **共有コアモジュール** (`lib/skills-core.js`)
   - 共通のスキル検出および解析ロジック
   - Codex 実装と OpenCode 実装の両方で使用

2. **プラットフォーム固有のラッパー**
   - Codex: CLI スクリプト (`.codex/superpowers-codex`)
   - OpenCode: プラグインモジュール (`.opencode/plugin/superpowers.js`)

3. **スキルディレクトリ**
   - Core: `~/.config/opencode/superpowers/skills/`（またはインストール先）
   - Personal: `~/.config/opencode/skills/`（core スキルを上書き）

### コード再利用戦略

`.codex/superpowers-codex` から共通機能を共有モジュールへ抽出する:

```javascript
// lib/skills-core.js
module.exports = {
  extractFrontmatter(filePath),      // Parse name + description from YAML
  findSkillsInDir(dir, maxDepth),    // Recursive SKILL.md discovery
  findAllSkills(dirs),                // Scan multiple directories
  resolveSkillPath(skillName, dirs), // Handle shadowing (personal > core)
  checkForUpdates(repoDir)           // Git fetch/status check
};
```

### スキル frontmatter 形式

現在の形式（`when_to_use` フィールドなし）:

```yaml
---
name: skill-name
description: Use when [condition] - [what it does]; [additional context]
---
```

## OpenCode プラグイン実装

### カスタムツール

**ツール 1: `use_skill`**

会話に特定のスキル内容を読み込む（Claude の Skill ツールに相当）。

```javascript
{
  name: 'use_skill',
  description: 'Load and read a specific skill to guide your work',
  schema: z.object({
    skill_name: z.string().describe('Name of skill (e.g., "superpowers:brainstorming")')
  }),
  execute: async ({ skill_name }) => {
    const { skillPath, content, frontmatter } = resolveAndReadSkill(skill_name);
    const skillDir = path.dirname(skillPath);

    return `# ${frontmatter.name}
# ${frontmatter.description}
# Supporting tools and docs are in ${skillDir}
# ============================================

${content}`;
  }
}
```

**ツール 2: `find_skills`**

利用可能なすべてのスキルをメタデータ付きで一覧表示する。

```javascript
{
  name: 'find_skills',
  description: 'List all available skills',
  schema: z.object({}),
  execute: async () => {
    const skills = discoverAllSkills();
    return skills.map(s =>
      `${s.namespace}:${s.name}
  ${s.description}
  Directory: ${s.directory}
`).join('\n');
  }
}
```

### セッション開始フック

新しいセッションが開始されたとき（`session.started` イベント）:

1. **using-superpowers の内容を注入**
   - using-superpowers スキルの全文
   - 必須ワークフローを確立する

2. **`find_skills` を自動実行**
   - 利用可能なスキルの完全な一覧を最初に表示する
   - それぞれのスキルについてスキルディレクトリを含める

3. **ツール対応付けの説明を注入**
   ```markdown
   **Tool Mapping for OpenCode:**
   When skills reference tools you don't have, substitute:
   - `TodoWrite` → `update_plan`
   - `Task` with subagents → Use OpenCode subagent system (@mention)
   - `Skill` tool → `use_skill` custom tool
   - Read, Write, Edit, Bash → Your native equivalents

   **Skill directories contain:**
   - Supporting scripts (run with bash)
   - Additional documentation (read with read tool)
   - Utilities specific to that skill
   ```

4. **更新確認**（非ブロッキング）
   - タイムアウト付きの高速な git fetch
   - 更新が利用可能なら通知

### プラグイン構造

```javascript
// .opencode/plugin/superpowers.js
const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const { z } = require('zod');

export const SuperpowersPlugin = async ({ client, directory, $ }) => {
  const superpowersDir = path.join(process.env.HOME, '.config/opencode/superpowers');
  const personalDir = path.join(process.env.HOME, '.config/opencode/skills');

  return {
    'session.started': async () => {
      const usingSuperpowers = await readSkill('using-superpowers');
      const skillsList = await findAllSkills();
      const toolMapping = getToolMappingInstructions();

      return {
        context: `${usingSuperpowers}\n\n${skillsList}\n\n${toolMapping}`
      };
    },

    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill',
        schema: z.object({
          skill_name: z.string()
        }),
        execute: async ({ skill_name }) => {
          // Implementation using skillsCore
        }
      },
      {
        name: 'find_skills',
        description: 'List all available skills',
        schema: z.object({}),
        execute: async () => {
          // Implementation using skillsCore
        }
      }
    ]
  };
};
```

## ファイル構成

```
superpowers/
├── lib/
│   └── skills-core.js           # NEW: Shared skill logic
├── .codex/
│   ├── superpowers-codex        # UPDATED: Use skills-core
│   ├── superpowers-bootstrap.md
│   └── INSTALL.md
├── .opencode/
│   ├── plugin/
│   │   └── superpowers.js       # NEW: OpenCode plugin
│   └── INSTALL.md               # NEW: Installation guide
└── skills/                       # Unchanged
```

## 実装計画

### フェーズ 1: 共有コアのリファクタリング

1. `lib/skills-core.js` を作成する
   - `.codex/superpowers-codex` から frontmatter 解析を抽出する
   - スキル検出ロジックを抽出する
   - パス解決（シャドーイング付き）を抽出する
   - `name` と `description` のみを使うよう更新する（`when_to_use` なし）

2. `.codex/superpowers-codex` を共有コア利用に更新する
   - `../lib/skills-core.js` から import する
   - 重複コードを削除する
   - CLI ラッパーロジックは維持する

3. Codex 実装が引き続き動作することをテストする
   - bootstrap コマンドを確認する
   - use-skill コマンドを確認する
   - find-skills コマンドを確認する

### フェーズ 2: OpenCode プラグインを構築する

1. `.opencode/plugin/superpowers.js` を作成する
   - `../../lib/skills-core.js` から共有コアを import する
   - プラグイン関数を実装する
   - カスタムツール（use_skill、find_skills）を定義する
   - session.started フックを実装する

2. `.opencode/INSTALL.md` を作成する
   - インストール手順
   - ディレクトリ設定
   - 設定ガイダンス

3. OpenCode 実装をテストする
   - セッション開始時の bootstrap を確認する
   - `use_skill` ツールが動作することを確認する
   - `find_skills` ツールが動作することを確認する
   - スキルディレクトリにアクセスできることを確認する

### フェーズ 3: ドキュメントと仕上げ

1. README を OpenCode 対応で更新する
2. メインドキュメントに OpenCode のインストールを追加する
3. RELEASE-NOTES を更新する
4. Codex と OpenCode の両方が正しく動作することをテストする

## 次のステップ

1. **分離されたワークスペースを作成する**（git worktrees を使用）
   - ブランチ: `feature/opencode-support`

2. **適用可能な箇所では TDD に従う**
   - 共有コア関数をテストする
   - スキルの検出と解析をテストする
   - 両プラットフォーム向けの統合テスト

3. **段階的に実装する**
   - フェーズ 1: 共有コアをリファクタリングし、Codex を更新する
   - 次に進む前に Codex が引き続き動作することを確認する
   - フェーズ 2: OpenCode プラグインを構築する
   - フェーズ 3: ドキュメントと仕上げ

4. **テスト戦略**
   - 実際の OpenCode インストールで手動テストを行う
   - スキルの読み込み、ディレクトリ、スクリプトが動作することを確認する
   - Codex と OpenCode を並べてテストする
   - ツールマッピングが正しく機能することを確認する

5. **PR とマージ**
   - 完全な実装で PR を作成する
   - クリーンな環境でテストする
   - main にマージする

## 利点

- **コード再利用**: スキル検出/解析の単一の信頼できる情報源
- **保守性**: バグ修正が両プラットフォームに適用される
- **拡張性**: 将来のプラットフォーム（Cursor、Windsurf など）を簡単に追加できる
- **ネイティブ統合**: OpenCode のプラグインシステムを正しく利用する
- **一貫性**: すべてのプラットフォームで同じスキル体験を提供する
