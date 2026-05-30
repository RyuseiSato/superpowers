# OpenCode サポート実装計画

> **エージェント型ワーカー向け:** 必須サブスキル: この計画をタスクごとに実装するには superpowers:executing-plans を使用すること。

**目標:** 既存の Codex 実装とコア機能を共有するネイティブな JavaScript プラグインにより、OpenCode.ai 向けの superpowers 完全対応を追加する。

**アーキテクチャ:** 共通のスキル検出/解析ロジックを `lib/skills-core.js` に抽出し、Codex がそれを使うようにリファクタリングし、その後でカスタムツールとセッションフックを備えたネイティブなプラグイン API を使って OpenCode プラグインを構築する。

**技術スタック:** Node.js, JavaScript, OpenCode Plugin API, Git worktrees

---

## フェーズ 1: 共有コアモジュールを作成する

### タスク 1: Frontmatter 解析を抽出する

**ファイル:**
- 作成: `lib/skills-core.js`
- 参照: `.codex/superpowers-codex`（40-74 行目）

**ステップ 1: extractFrontmatter 関数を含む lib/skills-core.js を作成する**

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

/**
 * Extract YAML frontmatter from a skill file.
 * Current format:
 * ---
 * name: skill-name
 * description: Use when [condition] - [what it does]
 * ---
 *
 * @param {string} filePath - Path to SKILL.md file
 * @returns {{name: string, description: string}}
 */
function extractFrontmatter(filePath) {
    try {
        const content = fs.readFileSync(filePath, 'utf8');
        const lines = content.split('\n');

        let inFrontmatter = false;
        let name = '';
        let description = '';

        for (const line of lines) {
            if (line.trim() === '---') {
                if (inFrontmatter) break;
                inFrontmatter = true;
                continue;
            }

            if (inFrontmatter) {
                const match = line.match(/^(\w+):\s*(.*)$/);
                if (match) {
                    const [, key, value] = match;
                    switch (key) {
                        case 'name':
                            name = value.trim();
                            break;
                        case 'description':
                            description = value.trim();
                            break;
                    }
                }
            }
        }

        return { name, description };
    } catch (error) {
        return { name: '', description: '' };
    }
}

module.exports = {
    extractFrontmatter
};
```

**ステップ 2: ファイルが作成されたことを確認する**

実行: `ls -l lib/skills-core.js`
期待結果: ファイルが存在する

**ステップ 3: コミット**

```bash
git add lib/skills-core.js
git commit -m "feat: create shared skills core module with frontmatter parser"
```

---

### タスク 2: スキル検出ロジックを抽出する

**ファイル:**
- 変更: `lib/skills-core.js`
- 参照: `.codex/superpowers-codex`（97-136 行目）

**ステップ 1: skills-core.js に findSkillsInDir 関数を追加する**

`module.exports` の前に追加:

```javascript
/**
 * Find all SKILL.md files in a directory recursively.
 *
 * @param {string} dir - Directory to search
 * @param {string} sourceType - 'personal' or 'superpowers' for namespacing
 * @param {number} maxDepth - Maximum recursion depth (default: 3)
 * @returns {Array<{path: string, name: string, description: string, sourceType: string}>}
 */
function findSkillsInDir(dir, sourceType, maxDepth = 3) {
    const skills = [];

    if (!fs.existsSync(dir)) return skills;

    function recurse(currentDir, depth) {
        if (depth > maxDepth) return;

        const entries = fs.readdirSync(currentDir, { withFileTypes: true });

        for (const entry of entries) {
            const fullPath = path.join(currentDir, entry.name);

            if (entry.isDirectory()) {
                // Check for SKILL.md in this directory
                const skillFile = path.join(fullPath, 'SKILL.md');
                if (fs.existsSync(skillFile)) {
                    const { name, description } = extractFrontmatter(skillFile);
                    skills.push({
                        path: fullPath,
                        skillFile: skillFile,
                        name: name || entry.name,
                        description: description || '',
                        sourceType: sourceType
                    });
                }

                // Recurse into subdirectories
                recurse(fullPath, depth + 1);
            }
        }
    }

    recurse(dir, 0);
    return skills;
}
```

**ステップ 2: module.exports を更新する**

exports 行を以下に置き換える:

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir
};
```

**ステップ 3: 構文を確認する**

実行: `node -c lib/skills-core.js`
期待結果: 出力なし（成功）

**ステップ 4: コミット**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill discovery function to core module"
```

---

### タスク 3: スキル解決ロジックを抽出する

**ファイル:**
- 変更: `lib/skills-core.js`
- 参照: `.codex/superpowers-codex`（212-280 行目）

**ステップ 1: resolveSkillPath 関数を追加する**

`module.exports` の前に追加:

```javascript
/**
 * Resolve a skill name to its file path, handling shadowing
 * (personal skills override superpowers skills).
 *
 * @param {string} skillName - Name like "superpowers:brainstorming" or "my-skill"
 * @param {string} superpowersDir - Path to superpowers skills directory
 * @param {string} personalDir - Path to personal skills directory
 * @returns {{skillFile: string, sourceType: string, skillPath: string} | null}
 */
function resolveSkillPath(skillName, superpowersDir, personalDir) {
    // Strip superpowers: prefix if present
    const forceSuperpowers = skillName.startsWith('superpowers:');
    const actualSkillName = forceSuperpowers ? skillName.replace(/^superpowers:/, '') : skillName;

    // Try personal skills first (unless explicitly superpowers:)
    if (!forceSuperpowers && personalDir) {
        const personalPath = path.join(personalDir, actualSkillName);
        const personalSkillFile = path.join(personalPath, 'SKILL.md');
        if (fs.existsSync(personalSkillFile)) {
            return {
                skillFile: personalSkillFile,
                sourceType: 'personal',
                skillPath: actualSkillName
            };
        }
    }

    // Try superpowers skills
    if (superpowersDir) {
        const superpowersPath = path.join(superpowersDir, actualSkillName);
        const superpowersSkillFile = path.join(superpowersPath, 'SKILL.md');
        if (fs.existsSync(superpowersSkillFile)) {
            return {
                skillFile: superpowersSkillFile,
                sourceType: 'superpowers',
                skillPath: actualSkillName
            };
        }
    }

    return null;
}
```

**ステップ 2: module.exports を更新する**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath
};
```

**ステップ 3: 構文を確認する**

実行: `node -c lib/skills-core.js`
期待結果: 出力なし

**ステップ 4: コミット**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill path resolution with shadowing support"
```

---

### タスク 4: 更新確認ロジックを抽出する

**ファイル:**
- 変更: `lib/skills-core.js`
- 参照: `.codex/superpowers-codex`（16-38 行目）

**ステップ 1: checkForUpdates 関数を追加する**

requires の後ろの先頭付近に追加:

```javascript
const { execSync } = require('child_process');
```

`module.exports` の前に追加:

```javascript
/**
 * Check if a git repository has updates available.
 *
 * @param {string} repoDir - Path to git repository
 * @returns {boolean} - True if updates are available
 */
function checkForUpdates(repoDir) {
    try {
        // Quick check with 3 second timeout to avoid delays if network is down
        const output = execSync('git fetch origin && git status --porcelain=v1 --branch', {
            cwd: repoDir,
            timeout: 3000,
            encoding: 'utf8',
            stdio: 'pipe'
        });

        // Parse git status output to see if we're behind
        const statusLines = output.split('\n');
        for (const line of statusLines) {
            if (line.startsWith('## ') && line.includes('[behind ')) {
                return true; // We're behind remote
            }
        }
        return false; // Up to date
    } catch (error) {
        // Network down, git error, timeout, etc. - don't block bootstrap
        return false;
    }
}
```

**ステップ 2: module.exports を更新する**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath,
    checkForUpdates
};
```

**ステップ 3: 構文を確認する**

実行: `node -c lib/skills-core.js`
期待結果: 出力なし

**ステップ 4: コミット**

```bash
git add lib/skills-core.js
git commit -m "feat: add git update checking to core module"
```

---

## フェーズ 2: Codex が共有コアを使うようにリファクタリングする

### タスク 5: Codex を更新して共有コアを import する

**ファイル:**
- 変更: `.codex/superpowers-codex`（先頭に import を追加）

**ステップ 1: import 文を追加する**

ファイル先頭の既存の requires の後（6 行目付近）に以下を追加:

```javascript
const skillsCore = require('../lib/skills-core');
```

**ステップ 2: 構文を確認する**

実行: `node -c .codex/superpowers-codex`
期待結果: 出力なし

**ステップ 3: コミット**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: import shared skills core in codex"
```

---

### タスク 6: extractFrontmatter をコア版に置き換える

**ファイル:**
- 変更: `.codex/superpowers-codex`（40-74 行目）

**ステップ 1: ローカルの extractFrontmatter 関数を削除する**

40-74 行目（extractFrontmatter 関数定義全体）を削除する。

**ステップ 2: すべての extractFrontmatter 呼び出しを更新する**

`extractFrontmatter(` を `skillsCore.extractFrontmatter(` にすべて検索置換する。

対象行の目安: 90 行目、310 行目

**ステップ 3: スクリプトが引き続き動作することを確認する**

実行: `.codex/superpowers-codex find-skills | head -20`
期待結果: スキル一覧が表示される

**ステップ 4: コミット**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared extractFrontmatter in codex"
```

---

### タスク 7: findSkillsInDir をコア版に置き換える

**ファイル:**
- 変更: `.codex/superpowers-codex`（おおよそ 97-136 行目）

**ステップ 1: ローカルの findSkillsInDir 関数を削除する**

`findSkillsInDir` 関数定義全体（おおよそ 97-136 行目）を削除する。

**ステップ 2: すべての findSkillsInDir 呼び出しを更新する**

`findSkillsInDir(` を `skillsCore.findSkillsInDir(` に置き換える。

**ステップ 3: スクリプトが引き続き動作することを確認する**

実行: `.codex/superpowers-codex find-skills | head -20`
期待結果: スキル一覧が表示される

**ステップ 4: コミット**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared findSkillsInDir in codex"
```

---

### タスク 8: checkForUpdates をコア版に置き換える

**ファイル:**
- 変更: `.codex/superpowers-codex`（おおよそ 16-38 行目）

**ステップ 1: ローカルの checkForUpdates 関数を削除する**

checkForUpdates 関数定義全体を削除する。

**ステップ 2: すべての checkForUpdates 呼び出しを更新する**

`checkForUpdates(` を `skillsCore.checkForUpdates(` に置き換える。

**ステップ 3: スクリプトが引き続き動作することを確認する**

実行: `.codex/superpowers-codex bootstrap | head -50`
期待結果: bootstrap 内容が表示される

**ステップ 4: コミット**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared checkForUpdates in codex"
```

---

## フェーズ 3: OpenCode プラグインを構築する

### タスク 9: OpenCode プラグインのディレクトリ構成を作成する

**ファイル:**
- 作成: `.opencode/plugin/superpowers.js`

**ステップ 1: ディレクトリを作成する**

実行: `mkdir -p .opencode/plugin`

**ステップ 2: 基本的なプラグインファイルを作成する**

```javascript
#!/usr/bin/env node

/**
 * Superpowers plugin for OpenCode.ai
 *
 * Provides custom tools for loading and discovering skills,
 * with automatic bootstrap on session start.
 */

const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const os = require('os');

const homeDir = os.homedir();
const superpowersSkillsDir = path.join(homeDir, '.config/opencode/superpowers/skills');
const personalSkillsDir = path.join(homeDir, '.config/opencode/skills');

/**
 * OpenCode plugin entry point
 */
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // Custom tools and hooks will go here
  };
};
```

**ステップ 3: ファイルが作成されたことを確認する**

実行: `ls -l .opencode/plugin/superpowers.js`
期待結果: ファイルが存在する

**ステップ 4: コミット**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: create opencode plugin scaffold"
```

---

### タスク 10: use_skill ツールを実装する

**ファイル:**
- 変更: `.opencode/plugin/superpowers.js`

**ステップ 1: use_skill ツール実装を追加する**

プラグインの return 文を以下に置き換える:

```javascript
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  // Import zod for schema validation
  const { z } = await import('zod');

  return {
    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill to guide your work. Skills contain proven workflows, mandatory processes, and expert techniques.',
        schema: z.object({
          skill_name: z.string().describe('Name of the skill to load (e.g., "superpowers:brainstorming" or "my-custom-skill")')
        }),
        execute: async ({ skill_name }) => {
          // Resolve skill path (handles shadowing: personal > superpowers)
          const resolved = skillsCore.resolveSkillPath(
            skill_name,
            superpowersSkillsDir,
            personalSkillsDir
          );

          if (!resolved) {
            return `Error: Skill "${skill_name}" not found.\n\nRun find_skills to see available skills.`;
          }

          // Read skill content
          const fullContent = fs.readFileSync(resolved.skillFile, 'utf8');
          const { name, description } = skillsCore.extractFrontmatter(resolved.skillFile);

          // Extract content after frontmatter
          const lines = fullContent.split('\n');
          let inFrontmatter = false;
          let frontmatterEnded = false;
          const contentLines = [];

          for (const line of lines) {
            if (line.trim() === '---') {
              if (inFrontmatter) {
                frontmatterEnded = true;
                continue;
              }
              inFrontmatter = true;
              continue;
            }

            if (frontmatterEnded || !inFrontmatter) {
              contentLines.push(line);
            }
          }

          const content = contentLines.join('\n').trim();
          const skillDirectory = path.dirname(resolved.skillFile);

          // Format output similar to Claude Code's Skill tool
          return `# ${name || skill_name}
# ${description || ''}
# Supporting tools and docs are in ${skillDirectory}
# ============================================

${content}`;
        }
      }
    ]
  };
};
```

**ステップ 2: 構文を確認する**

実行: `node -c .opencode/plugin/superpowers.js`
期待結果: 出力なし

**ステップ 3: コミット**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement use_skill tool for opencode"
```

---

### タスク 11: find_skills ツールを実装する

**ファイル:**
- 変更: `.opencode/plugin/superpowers.js`

**ステップ 1: tools 配列に find_skills ツールを追加する**

use_skill ツール定義の後、tools 配列を閉じる前に追加:

```javascript
      {
        name: 'find_skills',
        description: 'List all available skills in the superpowers and personal skill libraries.',
        schema: z.object({}),
        execute: async () => {
          // Find skills in both directories
          const superpowersSkills = skillsCore.findSkillsInDir(
            superpowersSkillsDir,
            'superpowers',
            3
          );
          const personalSkills = skillsCore.findSkillsInDir(
            personalSkillsDir,
            'personal',
            3
          );

          // Combine and format skills list
          const allSkills = [...personalSkills, ...superpowersSkills];

          if (allSkills.length === 0) {
            return 'No skills found. Install superpowers skills to ~/.config/opencode/superpowers/skills/';
          }

          let output = 'Available skills:\n\n';

          for (const skill of allSkills) {
            const namespace = skill.sourceType === 'personal' ? '' : 'superpowers:';
            const skillName = skill.name || path.basename(skill.path);

            output += `${namespace}${skillName}\n`;
            if (skill.description) {
              output += `  ${skill.description}\n`;
            }
            output += `  Directory: ${skill.path}\n\n`;
          }

          return output;
        }
      }
```

**ステップ 2: 構文を確認する**

実行: `node -c .opencode/plugin/superpowers.js`
期待結果: 出力なし

**ステップ 3: コミット**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement find_skills tool for opencode"
```

---

### タスク 12: セッション開始フックを実装する

**ファイル:**
- 変更: `.opencode/plugin/superpowers.js`

**ステップ 1: session.started フックを追加する**

tools 配列の後に以下を追加:

```javascript
    'session.started': async () => {
      // Read using-superpowers skill content
      const usingSuperpowersPath = skillsCore.resolveSkillPath(
        'using-superpowers',
        superpowersSkillsDir,
        personalSkillsDir
      );

      let usingSuperpowersContent = '';
      if (usingSuperpowersPath) {
        const fullContent = fs.readFileSync(usingSuperpowersPath.skillFile, 'utf8');
        // Strip frontmatter
        const lines = fullContent.split('\n');
        let inFrontmatter = false;
        let frontmatterEnded = false;
        const contentLines = [];

        for (const line of lines) {
          if (line.trim() === '---') {
            if (inFrontmatter) {
              frontmatterEnded = true;
              continue;
            }
            inFrontmatter = true;
            continue;
          }

          if (frontmatterEnded || !inFrontmatter) {
            contentLines.push(line);
          }
        }

        usingSuperpowersContent = contentLines.join('\n').trim();
      }

      // Tool mapping instructions
      const toolMapping = `
**Tool Mapping for OpenCode:**
When skills reference tools you don't have, substitute OpenCode equivalents:
- \`TodoWrite\` → \`update_plan\` (your planning/task tracking tool)
- \`Task\` tool with subagents → Use OpenCode's subagent system (@mention syntax or automatic dispatch)
- \`Skill\` tool → \`use_skill\` custom tool (already available)
- \`Read\`, \`Write\`, \`Edit\`, \`Bash\` → Use your native tools

**Skill directories contain supporting files:**
- Scripts you can run with bash tool
- Additional documentation you can read
- Utilities and helpers specific to that skill

**Skills naming:**
- Superpowers skills: \`superpowers:skill-name\` (from ~/.config/opencode/superpowers/skills/)
- Personal skills: \`skill-name\` (from ~/.config/opencode/skills/)
- Personal skills override superpowers skills when names match
`;

      // Check for updates (non-blocking)
      const hasUpdates = skillsCore.checkForUpdates(
        path.join(homeDir, '.config/opencode/superpowers')
      );

      const updateNotice = hasUpdates ?
        '\n\n⚠️ **Updates available!** Run `cd ~/.config/opencode/superpowers && git pull` to update superpowers.' :
        '';

      // Return context to inject into session
      return {
        context: `<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'use_skill' tool:**

${usingSuperpowersContent}

${toolMapping}${updateNotice}
</EXTREMELY_IMPORTANT>`
      };
    }
```

**ステップ 2: 構文を確認する**

実行: `node -c .opencode/plugin/superpowers.js`
期待結果: 出力なし

**ステップ 3: コミット**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement session.started hook for opencode"
```

---

## フェーズ 4: ドキュメント

### タスク 13: OpenCode インストールガイドを作成する

**ファイル:**
- 作成: `.opencode/INSTALL.md`

**ステップ 1: インストールガイドを作成する**

```markdown
# Installing Superpowers for OpenCode

## Prerequisites

- [OpenCode.ai](https://opencode.ai) installed
- Node.js installed
- Git installed

## Installation Steps

### 1. Install Superpowers Skills

```bash
# Clone superpowers skills to OpenCode config directory
mkdir -p ~/.config/opencode/superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. Install the Plugin

The plugin is included in the superpowers repository you just cloned.

OpenCode will automatically discover it from:
- `~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`

Or you can link it to the project-local plugin directory:

```bash
# In your OpenCode project
mkdir -p .opencode/plugin
ln -s ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js .opencode/plugin/superpowers.js
```

### 3. Restart OpenCode

Restart OpenCode to load the plugin. On the next session, you should see:

```
You have superpowers.
```

## Usage

### Finding Skills

Use the `find_skills` tool to list all available skills:

```
use find_skills tool
```

### Loading a Skill

Use the `use_skill` tool to load a specific skill:

```
use use_skill tool with skill_name: "superpowers:brainstorming"
```

### Personal Skills

Create your own skills in `~/.config/opencode/skills/`:

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

Create `~/.config/opencode/skills/my-skill/SKILL.md`:

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

Personal skills override superpowers skills with the same name.

## Updating

```bash
cd ~/.config/opencode/superpowers
git pull
```

## Troubleshooting

### Plugin not loading

1. Check plugin file exists: `ls ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`
2. Check OpenCode logs for errors
3. Verify Node.js is installed: `node --version`

### Skills not found

1. Verify skills directory exists: `ls ~/.config/opencode/superpowers/skills`
2. Use `find_skills` tool to see what's discovered
3. Check file structure: each skill should have a `SKILL.md` file

### Tool mapping issues

When a skill references a Claude Code tool you don't have:
- `TodoWrite` → use `update_plan`
- `Task` with subagents → use `@mention` syntax to invoke OpenCode subagents
- `Skill` → use `use_skill` tool
- File operations → use your native tools

## Getting Help

- Report issues: https://github.com/obra/superpowers/issues
- Documentation: https://github.com/obra/superpowers
```

**ステップ 2: ファイルが作成されたことを確認する**

実行: `ls -l .opencode/INSTALL.md`
期待結果: ファイルが存在する

**ステップ 3: コミット**

```bash
git add .opencode/INSTALL.md
git commit -m "docs: add opencode installation guide"
```

---

### タスク 14: メイン README を更新する

**ファイル:**
- 変更: `README.md`

**ステップ 1: OpenCode セクションを追加する**

対応プラットフォームに関するセクション（ファイル内で "Codex" を検索）を見つけ、その後に以下を追加:

```markdown
### OpenCode

Superpowers works with [OpenCode.ai](https://opencode.ai) through a native JavaScript plugin.

**Installation:** See [.opencode/INSTALL.md](.opencode/INSTALL.md)

**Features:**
- Custom tools: `use_skill` and `find_skills`
- Automatic session bootstrap
- Personal skills with shadowing
- Supporting files and scripts access
```

**ステップ 2: 書式を確認する**

実行: `grep -A 10 "### OpenCode" README.md`
期待結果: 追加したセクションが表示される

**ステップ 3: コミット**

```bash
git add README.md
git commit -m "docs: add opencode support to readme"
```

---

### タスク 15: リリースノートを更新する

**ファイル:**
- 変更: `RELEASE-NOTES.md`

**ステップ 1: OpenCode サポートの項目を追加する**

ファイル先頭（ヘッダーの後）に以下を追加:

```markdown
## [Unreleased]

### Added

- **OpenCode Support**: Native JavaScript plugin for OpenCode.ai
  - Custom tools: `use_skill` and `find_skills`
  - Automatic session bootstrap with tool mapping instructions
  - Shared core module (`lib/skills-core.js`) for code reuse
  - Installation guide in `.opencode/INSTALL.md`

### Changed

- **Refactored Codex Implementation**: Now uses shared `lib/skills-core.js` module
  - Eliminates code duplication between Codex and OpenCode
  - Single source of truth for skill discovery and parsing

---

```

**ステップ 2: 書式を確認する**

実行: `head -30 RELEASE-NOTES.md`
期待結果: 新しいセクションが表示される

**ステップ 3: コミット**

```bash
git add RELEASE-NOTES.md
git commit -m "docs: add opencode support to release notes"
```

---

## フェーズ 5: 最終確認

### タスク 16: Codex が引き続き動作することをテストする

**ファイル:**
- テスト: `.codex/superpowers-codex`

**ステップ 1: find-skills コマンドをテストする**

実行: `.codex/superpowers-codex find-skills | head -20`
期待結果: 名前と説明付きでスキル一覧が表示される

**ステップ 2: use-skill コマンドをテストする**

実行: `.codex/superpowers-codex use-skill superpowers:brainstorming | head -20`
期待結果: brainstorming スキルの内容が表示される

**ステップ 3: bootstrap コマンドをテストする**

実行: `.codex/superpowers-codex bootstrap | head -30`
期待結果: 説明付きの bootstrap 内容が表示される

**ステップ 4: すべてのテストに通ったら、成功を記録する**

コミット不要 - これは確認のみ。

---

### タスク 17: ファイル構成を確認する

**ファイル:**
- 確認: すべての新規ファイルが存在すること

**ステップ 1: すべてのファイルが作成されたことを確認する**

実行:
```bash
ls -l lib/skills-core.js
ls -l .opencode/plugin/superpowers.js
ls -l .opencode/INSTALL.md
```

期待結果: すべてのファイルが存在する

**ステップ 2: ディレクトリ構成を確認する**

実行: `tree -L 2 .opencode/`（tree が利用できない場合は `find .opencode -type f`）
期待結果:
```
.opencode/
├── INSTALL.md
└── plugin/
    └── superpowers.js
```

**ステップ 3: 構成が正しければ続行する**

コミット不要 - これは確認のみ。

---

### タスク 18: 最終コミットと要約

**ファイル:**
- 確認: `git status`

**ステップ 1: git status を確認する**

実行: `git status`
期待結果: 作業ツリーがクリーンで、すべての変更がコミット済み

**ステップ 2: コミットログを確認する**

実行: `git log --oneline -20`
期待結果: この実装に関するすべてのコミットが表示される

**ステップ 3: サマリードキュメントを作成する**

以下を示す完了サマリーを作成する:
- 作成したコミット総数
- 作成したファイル: `lib/skills-core.js`, `.opencode/plugin/superpowers.js`, `.opencode/INSTALL.md`
- 変更したファイル: `.codex/superpowers-codex`, `README.md`, `RELEASE-NOTES.md`
- 実施したテスト: Codex コマンドを確認済み
- 準備完了: 実際の OpenCode インストールでのテスト

**ステップ 4: 完了を報告する**

ユーザーにサマリーを提示し、以下を提案する:
1. リモートへ push する
2. pull request を作成する
3. 実際の OpenCode インストールでテストする（OpenCode のインストールが必要）

---

## テストガイド（手動 - OpenCode が必要）

これらの手順には OpenCode のインストールが必要であり、自動実装の対象外です:

1. **スキルをインストール**: `.opencode/INSTALL.md` に従う
2. **OpenCode セッションを開始**: bootstrap が表示されることを確認する
3. **find_skills をテスト**: 利用可能なすべてのスキルが一覧表示されるはず
4. **use_skill をテスト**: スキルを読み込み、内容が表示されることを確認する
5. **サポートファイルをテスト**: スキルディレクトリのパスにアクセスできることを確認する
6. **個人スキルをテスト**: 個人スキルを作成し、core をシャドーできることを確認する
7. **ツール対応付けをテスト**: TodoWrite → update_plan の対応付けが機能することを確認する

## 成功条件

- [ ] `lib/skills-core.js` がすべてのコア関数を備えて作成されている
- [ ] `.codex/superpowers-codex` が共有コアを使うようにリファクタリングされている
- [ ] Codex コマンドが引き続き動作する（find-skills, use-skill, bootstrap）
- [ ] `.opencode/plugin/superpowers.js` がツールとフック付きで作成されている
- [ ] インストールガイドが作成されている
- [ ] README と RELEASE-NOTES が更新されている
- [ ] すべての変更がコミットされている
- [ ] 作業ツリーがクリーンである
