# Skill authoring best practices

> Claude が見つけてうまく使える、効果的な Skills の書き方を学びましょう。

良い Skills は、簡潔で、よく構造化され、実際の利用でテストされています。このガイドは、Claude が見つけて効果的に使える Skills を書くための実践的な作成判断を提供します。

Skills の仕組みに関する概念的背景は、[Skills overview](/en/docs/agents-and-tools/agent-skills/overview) を参照してください。

## Core principles

### 簡潔さが重要

[context window](https://platform.claude.com/docs/en/build-with-claude/context-windows) は公共財です。あなたの Skill は、Claude が知る必要のある他のすべてと context window を共有します。たとえば:

* system prompt
* 会話履歴
* 他の Skills の metadata
* 実際のリクエスト

Skill 内のすべての token に即時コストがあるわけではありません。起動時に事前ロードされるのは、すべての Skills の metadata（name と description）だけです。Claude は Skill が関係したときだけ SKILL.md を読み、追加ファイルも必要に応じて読みます。それでも SKILL.md を簡潔に保つことは重要です。いったん Claude がそれを読み込めば、すべての token が会話履歴や他の context と競合するからです。

**デフォルト前提**: Claude はすでにとても賢い

Claude がまだ持っていない文脈だけを追加してください。各情報片について次を自問します:

* "Does Claude really need this explanation?"
* "Can I assume Claude knows this?"
* "Does this paragraph justify its token cost?"

**良い例: 簡潔**（およそ 50 tokens）:

````markdown  theme={null}
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**悪い例: 冗長すぎる**（およそ 150 tokens）:

```markdown  theme={null}
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available for PDF processing, but we
recommend pdfplumber because it's easy to use and handles most cases well.
First, you'll need to install it using pip. Then you can use the code below...
```

簡潔な版は、Claude が PDF とは何か、ライブラリがどう機能するかを知っている前提で書かれています。

### 適切な自由度を設定する

タスクの壊れやすさと変動性に応じて、具体性のレベルを合わせてください。

**自由度が高い**（テキスト中心の指示）:

使う場面:

* 複数のアプローチが妥当
* 判断が文脈に依存する
* ヒューリスティクスが手法を導く

例:

```markdown  theme={null}
## Code review process

1. Analyze the code structure and organization
2. Check for potential bugs or edge cases
3. Suggest improvements for readability and maintainability
4. Verify adherence to project conventions
```

**自由度が中程度**（疑似コードまたは引数付き script）:

使う場面:

* 好ましいパターンがある
* 多少の variation を許容できる
* 設定によって挙動が変わる

例:

````markdown  theme={null}
## Generate report

Use this template and customize as needed:

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
    # Optionally include visualizations
```
````

**自由度が低い**（具体的な script、引数ほぼなし）:

使う場面:

* 操作が壊れやすくエラーを起こしやすい
* 一貫性が重要
* 特定の順序を守る必要がある

例:

````markdown  theme={null}
## Database migration

Run exactly this script:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

**たとえ:** Claude を道を進むロボットだと考えてください:

* **両側が崖の狭い橋**: 安全な進み方は 1 つだけ。具体的な guardrails と正確な指示を与える（自由度低）。例: 正確な順序で実行しなければならない database migration。
* **危険のない開けた野原**: 多くの道が成功に通じる。一般的な方向だけ示し、最適な経路は Claude に任せる（自由度高）。例: 最善の進め方が文脈で決まる code review。

### 使う予定のすべての model でテストする

Skills は model への追加物として働くため、有効性は基盤 model に依存します。使う予定のすべての model で Skill をテストしてください。

**model ごとのテスト観点:**

* **Claude Haiku**（高速・経済的）: Skill は十分なガイダンスを与えているか？
* **Claude Sonnet**（バランス型）: Skill は明確で効率的か？
* **Claude Opus**（高い推論力）: Skill は過剰説明を避けているか？

Opus で完璧に機能するものも、Haiku には詳細がもっと必要かもしれません。複数 model で使うなら、すべてでうまく働く指示を目指してください。

## Skill structure

<Note>
  **YAML Frontmatter**: SKILL.md の frontmatter には次の 2 項目が必要です:

  * `name` - Skill の人間可読名（最大 64 文字）
  * `description` - Skill が何をするか、いつ使うかを 1 行で説明（最大 1024 文字）

  完全な Skill 構造の詳細は、[Skills overview](/en/docs/agents-and-tools/agent-skills/overview#skill-structure) を参照してください。
</Note>

### 命名規則

Skills を参照・議論しやすくするため、一貫した命名パターンを使ってください。**gerund form**（動詞 + -ing）を Skill 名に使うことを推奨します。これにより、その Skill が提供する活動や能力が明確になります。

**良い命名例（gerund form）:**

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**許容できる代替:**

* 名詞句: "PDF Processing", "Spreadsheet Analysis"
* 行動指向: "Process PDFs", "Analyze Spreadsheets"

**避けること:**

* 曖昧な名前: "Helper", "Utils", "Tools"
* 汎用的すぎるもの: "Documents", "Data", "Files"
* skill collection 内で一貫しないパターン

一貫した命名による利点:

* documentation や会話で Skills を参照しやすい
* Skill が何をするか一目で分かる
* 複数 Skills の整理と検索がしやすい
* 専門的で統一感のある skill library を維持できる

### 効果的な description の書き方

`description` フィールドは Skill discovery を担い、Skill が何をするかと、いつ使うかの両方を含めるべきです。

<Warning>
  **必ず三人称で書いてください。** description は system prompt に注入されるため、人称が一貫していないと discovery 問題を引き起こします。

  * **Good:** "Processes Excel files and generates reports"
  * **Avoid:** "I can help you process Excel files"
  * **Avoid:** "You can use this to process Excel files"
</Warning>

**具体的に、かつ key terms を含めてください。** Skill が何をするかと、それを使うべき具体的トリガー/文脈の両方を含めます。

各 Skill には description フィールドが 1 つだけあります。description は skill selection にとって非常に重要です。Claude は、100+ 個あるかもしれない Skills の中から適切なものを選ぶためにこれを使います。description は Claude が選ぶべきタイミングを判断できるだけの詳細を与え、SKILL.md の残り部分が実装詳細を提供します。

効果的な例:

**PDF Processing skill:**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel Analysis skill:**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill:**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

次のような曖昧な description は避けてください:

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### Progressive disclosure patterns

SKILL.md は必要に応じて詳細資料を指し示す overview として機能します。オンボーディングガイドの目次のようなものです。progressive disclosure がどう機能するかは、overview の [How Skills work](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work) を参照してください。

**実践的ガイダンス:**

* 最適な性能のため、SKILL.md 本文は 500 行未満に保つ
* この制限に近づいたら内容を別ファイルに分ける
* 以下のパターンを使って、指示、コード、リソースを効果的に整理する

#### 視覚的 overview: 単純から複雑へ

基本的な Skill は、metadata と指示を含む SKILL.md だけで始まります:

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

Skill が大きくなるにつれ、Claude が必要なときだけ読み込む追加コンテンツを束ねられます:

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完全な Skill directory structure は次のようになります:

```
pdf/
├── SKILL.md              # Main instructions (loaded when triggered)
├── FORMS.md              # Form-filling guide (loaded as needed)
├── reference.md          # API reference (loaded as needed)
├── examples.md           # Usage examples (loaded as needed)
└── scripts/
    ├── analyze_form.py   # Utility script (executed, not loaded)
    ├── fill_form.py      # Form filling script
    └── validate.py       # Validation script
```

#### Pattern 1: reference 付き high-level guide

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns
````

Claude は必要になったときだけ FORMS.md、REFERENCE.md、EXAMPLES.md を読み込みます。

#### Pattern 2: ドメイン別の構成

複数ドメインを持つ Skill では、無関係な context を読み込まないようドメインごとに内容を分けます。ユーザーが sales metrics を尋ねたとき、Claude に必要なのは sales 関連 schema であり、finance や marketing の data ではありません。こうすると token 使用量を抑え、context を集中させられます。

```
bigquery-skill/
├── SKILL.md (overview and navigation)
└── reference/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    ├── product.md (API usage, features)
    └── marketing.md (campaigns, attribution)
```

````markdown SKILL.md theme={null}
# BigQuery Data Analysis

## Available datasets

**Finance**: Revenue, ARR, billing → See [reference/finance.md](reference/finance.md)
**Sales**: Opportunities, pipeline, accounts → See [reference/sales.md](reference/sales.md)
**Product**: API usage, features, adoption → See [reference/product.md](reference/product.md)
**Marketing**: Campaigns, attribution, email → See [reference/marketing.md](reference/marketing.md)

## Quick search

Find specific metrics using grep:

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### Pattern 3: 条件付き詳細

基本内容を見せ、上級内容へリンクします:

```markdown  theme={null}
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

Claude はユーザーがその機能を必要としたときだけ REDLINING.md や OOXML.md を読みます。

### 深い入れ子の reference を避ける

Claude は、参照されたファイルからさらに参照されているファイルを部分的にしか読まないことがあります。入れ子の reference に遭遇すると、全文を読む代わりに `head -100` のようなコマンドで preview し、不完全な情報になることがあります。

**reference は SKILL.md から 1 階層だけに保ってください。** 必要になったとき Claude が完全なファイルを読むよう、すべての reference file は SKILL.md から直接リンクされているべきです。

**悪い例: 深すぎる**:

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here's the actual information...
```

**良い例: 1 階層だけ**:

```markdown  theme={null}
# SKILL.md

**Basic usage**: [instructions in SKILL.md]
**Advanced features**: See [advanced.md](advanced.md)
**API reference**: See [reference.md](reference.md)
**Examples**: See [examples.md](examples.md)
```

### 長い reference file は目次付きにする

100 行を超える reference file には先頭に table of contents を付けてください。Claude が部分読みするときでも利用可能な情報の全体像を把握できるようになります。

**例:**

```markdown  theme={null}
# API Reference

## Contents
- Authentication and setup
- Core methods (create, read, update, delete)
- Advanced features (batch operations, webhooks)
- Error handling patterns
- Code examples

## Authentication and setup
...

## Core methods
...
```

これにより Claude は必要に応じて全文を読むか、特定セクションへ移動できます。

この filesystem ベースの architecture が progressive disclosure をどう実現するかは、後述の Advanced セクションにある [Runtime environment](#runtime-environment) を参照してください。

## Workflows and feedback loops

### 複雑な task には workflow を使う

複雑な操作は、明確で順序立った step に分解してください。特に複雑な workflow では、Claude が自分の返答にコピーして進行中にチェックできる checklist を与えるとよいです。

**Example 1: Research synthesis workflow**（コード不要の Skills 向け）:

````markdown  theme={null}
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1: Read all source documents**

Review each document in the `sources/` directory. Note the main arguments and supporting evidence.

**Step 2: Identify key themes**

Look for patterns across sources. What themes appear repeatedly? Where do sources agree or disagree?

**Step 3: Cross-reference claims**

For each major claim, verify it appears in the source material. Note which source supports each point.

**Step 4: Create structured summary**

Organize findings by theme. Include:
- Main claim
- Supporting evidence from sources
- Conflicting viewpoints (if any)

**Step 5: Verify citations**

Check that every claim references the correct source document. If citations are incomplete, return to Step 3.
````

この例は、コードを必要としない分析 task に workflow がどう適用されるかを示しています。checklist パターンは、複雑で複数段階のあらゆる process に使えます。

**Example 2: PDF form filling workflow**（コードを含む Skills 向け）:

````markdown  theme={null}
## PDF form filling workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1: Analyze the form**

Run: `python scripts/analyze_form.py input.pdf`

This extracts form fields and their locations, saving to `fields.json`.

**Step 2: Create field mapping**

Edit `fields.json` to add values for each field.

**Step 3: Validate mapping**

Run: `python scripts/validate_fields.py fields.json`

Fix any validation errors before continuing.

**Step 4: Fill the form**

Run: `python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5: Verify output**

Run: `python scripts/verify_output.py output.pdf`

If verification fails, return to Step 2.
````

明確な step は、Claude が重要な validation を飛ばすのを防ぎます。checklist は、Claude とあなたの双方が multi-step workflow の進捗を追うのに役立ちます。

### feedback loop を実装する

**一般的なパターン**: validator を実行 → エラー修正 → 繰り返す

このパターンは出力品質を大きく改善します。

**Example 1: Style guide compliance**（コード不要の Skills 向け）:

```markdown  theme={null}
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

これは、script ではなく reference document を使った validation loop パターンです。"validator" は STYLE_GUIDE.md であり、Claude は読み比べることでチェックを行います。

**Example 2: Document editing process**（コードを含む Skills 向け）:

```markdown  theme={null}
## Document editing process

1. Make your edits to `word/document.xml`
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails:
   - Review the error message carefully
   - Fix the issues in the XML
   - Run validation again
4. **Only proceed when validation passes**
5. Rebuild: `python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. Test the output document
```

validation loop はエラーを早期に捕捉します。

## Content guidelines

### 時間依存の情報を避ける

古くなる情報を含めないでください:

**悪い例: 時間依存**（やがて誤りになる）:

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**良い例**（"old patterns" セクションを使う）:

```markdown  theme={null}
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

old patterns セクションは、主内容を散らかさずに履歴的文脈を提供します。

### 用語を一貫させる

Skill 全体で 1 つの用語を選び、統一して使ってください:

**Good - 一貫している:**

* 常に "API endpoint"
* 常に "field"
* 常に "extract"

**Bad - 一貫していない:**

* "API endpoint", "URL", "API route", "path" を混在させる
* "field", "box", "element", "control" を混在させる
* "extract", "pull", "get", "retrieve" を混在させる

一貫性は、Claude が指示を理解して従う助けになります。

## Common patterns

### Template pattern

出力形式の template を用意します。厳密さのレベルは必要性に合わせてください。

**厳格な要件向け**（API response や data format など）:

````markdown  theme={null}
## Report structure

ALWAYS use this exact template structure:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

**柔軟なガイダンス向け**（適応が有益な場合）:

````markdown  theme={null}
## Report structure

Here is a sensible default format, but use your best judgment based on the analysis:

```markdown
# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Tailor to the specific context]
```

Adjust sections as needed for the specific analysis type.
````

### Examples pattern

出力品質が例を見ることに強く依存する Skills では、通常の prompting と同じように input/output ペアを示します:

````markdown  theme={null}
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output:
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**Example 3:**
Input: Updated dependencies and refactored error handling
Output:
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

Follow this style: type(scope): brief description, then detailed explanation.
````

例は、説明だけよりも望ましいスタイルと詳細レベルを Claude に明確に伝えます。

### Conditional workflow pattern

判断ポイントを通して Claude を導きます:

```markdown  theme={null}
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

<Tip>
  workflow が大きく複雑になり step が多くなるなら、別ファイルに分離し、その task で適切なファイルを読むよう Claude に指示することを検討してください。
</Tip>

## Evaluation and iteration

### 先に evaluations を作る

**長い documentation を書く前に evaluations を作成してください。** これにより、想像上の問題ではなく実在の問題を Skill が解決するようになります。

**evaluation-driven development:**

1. **gaps を見つける**: Skill なしで代表的タスクを Claude に実行させ、具体的な失敗や不足文脈を記録する
2. **evaluations を作る**: それらの gaps を試す 3 つのシナリオを作る
3. **baseline を確立する**: Skill なしでの Claude の性能を測る
4. **最小限の instructions を書く**: gap を埋め、evaluation を通すのに十分な内容だけを書く
5. **反復する**: evaluation を実行し、baseline と比較し、改善する

この方法により、将来必要かもしれないと想像した要件ではなく、実際の問題を解くことが保証されます。

**evaluation の構造:**

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  この例は、簡単な testing rubric を使う data-driven evaluation を示しています。現時点では、これらの evaluation を実行する組み込み手段は提供していません。ユーザーは独自の evaluation system を作れます。Skill の有効性を測る真実の源は evaluation です。
</Note>

### Claude と反復的に Skills を開発する

最も効果的な Skill 開発プロセスには Claude 自身を使います。1 つの Claude インスタンス（"Claude A"）と協力して Skill を作り、それを別のインスタンス（"Claude B"）が使います。Claude A は指示設計と改善を手伝い、Claude B は実タスクでそれをテストします。これは、Claude models が効果的な agent 指示の書き方と、agent が必要とする情報の両方を理解しているためです。

**新しい Skill を作るとき:**

1. **Skill なしで task を完了する**: 通常の prompting を使って Claude A と問題に取り組む。作業中、文脈、好み、手順知識を自然に渡すことになる。繰り返し与えている情報に注目する。

2. **再利用可能なパターンを特定する**: task 完了後、今後の類似 task でも役立つ文脈が何だったかを特定する。

   **例**: BigQuery analysis を一緒に進めたなら、table 名、field 定義、filtering rule（例: "always exclude test accounts"）、よくある query パターンなどを渡していたかもしれません。

3. **Claude A に Skill を作らせる**: "Create a Skill that captures this BigQuery analysis pattern we just used. Include the table schemas, naming conventions, and the rule about filtering test accounts."

   <Tip>
     Claude models は Skill format と structure をネイティブに理解しています。Claude に Skills 作成を手伝わせるために特別な system prompt や "writing skills" skill は必要ありません。単に Skill を作るよう頼めば、適切な frontmatter と body を備えた SKILL.md を生成します。
   </Tip>

4. **簡潔さをレビューする**: Claude A が不要な説明を足していないか確認する。例: "Remove the explanation about what win rate means - Claude already knows that."

5. **情報アーキテクチャを改善する**: Claude A に内容をより効果的に整理させる。例: "Organize this so the table schema is in a separate reference file. We might add more tables later."

6. **類似タスクでテストする**: Skill を読み込んだ新しい Claude B で、関連ユースケースにその Skill を使う。Claude B が正しい情報を見つけ、ルールを正しく適用し、task をうまく処理できるか観察する。

7. **観察に基づいて反復する**: Claude B が苦戦したり何かを見落としたら、具体例を持って Claude A に戻る。例: "When Claude used this Skill, it forgot to filter by date for Q4. Should we add a section about date filtering patterns?"

**既存 Skills を改善するとき:**

Skill を改善するときも同じ階層パターンを続けます。次を行き来します:

* **Claude A と作業する**（Skill 改善を手伝う expert）
* **Claude B でテストする**（Skill を使って実際の仕事をする agent）
* **Claude B の挙動を観察し**、その洞察を Claude A に持ち帰る

1. **実ワークフローで Skill を使う**: Skill を読み込んだ Claude B に、テストシナリオではなく実タスクを与える

2. **Claude B の挙動を観察する**: 苦戦した点、うまくいった点、予想外の選択をした点を記録する

   **観察例**: "When I asked Claude B for a regional sales report, it wrote the query but forgot to filter out test accounts, even though the Skill mentions this rule."

3. **改善のため Claude A に戻る**: 現在の SKILL.md を共有し、観察内容を説明する。例: "I noticed Claude B forgot to filter test accounts when I asked for a regional report. The Skill mentions filtering, but maybe it's not prominent enough?"

4. **Claude A の提案をレビューする**: ルールを目立たせるための再構成、"always filter" より "MUST filter" のような強い表現、workflow section の再構成などを提案するかもしれません。

5. **変更を適用して再テストする**: Claude A の改善を Skill に反映し、同様の要求で再び Claude B でテストする

6. **利用に基づいて繰り返す**: 新しいシナリオに出会うたびに、この observe-refine-test サイクルを続ける。各反復は、想定ではなく観察された agent 挙動に基づいて Skill を改善します。

**チームのフィードバックを集める:**

1. teammate と Skills を共有し、その使い方を観察する
2. 次を尋ねる: 期待どおりに Skill は発火するか？ 指示は明確か？ 何が足りないか？
3. 自分の利用パターンの blind spot を補うようにフィードバックを取り込む

**この方法が効く理由:** Claude A は agent の必要を理解し、あなたはドメイン知識を提供し、Claude B は実利用を通じて gaps を明らかにし、反復改善が仮定ではなく観察された挙動に基づいて Skills をよくしていくからです。

### Claude が Skills をどう辿るか観察する

Skills を反復改善する際には、Claude が実際にそれらをどう使うかに注意してください。注目すべき点:

* **予想外の探索経路**: 予想しなかった順でファイルを読むか？ それは構成が思ったほど直感的ではないサインかもしれない
* **見逃されたつながり**: 重要ファイルへの reference を辿れないか？ link をもっと明示的または目立つ形にする必要があるかもしれない
* **特定セクションへの過度な依存**: 同じ file を繰り返し読むなら、その内容は main SKILL.md に入れるべきかもしれない
* **無視される内容**: bundled file に一度もアクセスしないなら、それは不要か main instructions でのシグナルが弱い可能性がある

仮定ではなく、こうした観察に基づいて反復してください。Skill metadata の `name` と `description` は特に重要です。Claude は現在の task に対して Skill を発火するかどうかをこれで判断します。Skill が何をするのか、いつ使うべきなのかを明確に書いてください。

## Anti-patterns to avoid

### Windows-style path を避ける

file path は Windows 上でも常に forward slash を使ってください:

* ✓ **Good**: `scripts/helper.py`, `reference/guide.md`
* ✗ **Avoid**: `scripts\helper.py`, `reference\guide.md`

Unix-style path は全プラットフォームで動きますが、Windows-style path は Unix 系でエラーになります。

### 選択肢を出しすぎない

必要がない限り複数アプローチを並べないでください:

````markdown  theme={null}
**Bad example: Too many choices** (confusing):
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**Good example: Provide a default** (with escape hatch):
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## Advanced: executable code を含む Skills

以下のセクションは executable script を含む Skills に焦点を当てています。markdown instructions だけの Skill なら、[Checklist for effective Skills](#checklist-for-effective-skills) へ進んでください。

### 投げずに解く

Skills のために script を書くときは、Claude に丸投げせず error condition を処理してください。

**良い例: エラーを明示的に処理する**:

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**悪い例: Claude に丸投げする**:

```python  theme={null}
def process_file(path):
    # Just fail and let Claude figure it out
    return open(path).read()
```

設定パラメータも "voodoo constants"（Ousterhout's law）にならないよう、根拠を示して文書化すべきです。適切な値を自分が分からないなら、Claude はどう判断すればよいのでしょうか？

**良い例: 自己説明的**:

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**悪い例: マジックナンバー**:

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### utility script を提供する

たとえ Claude が script を書けるとしても、用意済み script には利点があります:

**utility script の利点:**

* 生成コードより信頼性が高い
* token を節約できる（code を context に含める必要がない）
* 時間を節約できる（コード生成が不要）
* 利用ごとの一貫性を保証できる

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上の図は、executable script が instruction file と並んでどう機能するかを示しています。instruction file（forms.md）が script を参照し、Claude はその内容を context に読み込まずに実行できます。

**重要な区別:** 指示では、Claude が次のどちらをすべきか明確にしてください:

* **script を実行する**（最も一般的）: "Run `analyze_form.py` to extract fields"
* **reference として読む**（複雑なロジック向け）: "See `analyze_form.py` for the field extraction algorithm"

多くの utility script では、execution の方が信頼性・効率の両面で優れています。script execution の仕組みは後述の [Runtime environment](#runtime-environment) を参照してください。

**例:**

````markdown  theme={null}
## Utility scripts

**analyze_form.py**: Extract all form fields from PDF

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

Output format:
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**: Check for overlapping bounding boxes

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**: Apply field values to PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### visual analysis を使う

入力を画像として描画できるなら、Claude にそれを解析させてください:

````markdown  theme={null}
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. Claude can see field locations and types visually
````

<Note>
  この例では `pdf_to_images.py` script を自分で書く必要があります。
</Note>

Claude の vision capabilities はレイアウトや構造の理解に役立ちます。

### 検証可能な中間出力を作る

Claude が複雑で open-ended な task を行うと、ミスをすることがあります。"plan-validate-execute" パターンは、まず Claude に structured format で plan を作らせ、その plan を script で検証してから実行することで、エラーを早期に捕捉します。

**例:** spreadsheet に基づいて PDF 内の 50 個の form field を更新させたいとします。validation なしだと、Claude は存在しない field を参照したり、衝突する値を作ったり、必須 field を見落としたり、更新を誤ったりするかもしれません。

**解決策:** 上述の workflow pattern（PDF form filling）を使いつつ、変更適用前に検証される中間 `changes.json` file を加えます。workflow は analyze → **create plan file** → **validate plan** → execute → verify となります。

**このパターンが効く理由:**

* **エラーを早期に捕捉できる**: validation が変更適用前に問題を見つける
* **機械的に検証できる**: script が客観的な検証を提供する
* **可逆な planning**: Claude は原本に触れず plan を反復できる
* **明確なデバッグ**: エラーメッセージが具体的問題を指し示す

**使う場面**: batch operations、破壊的変更、複雑な validation rule、高リスク操作。

**実装 tip:** validation script は、"Field 'signature_date' not found. Available fields: customer_name, order_total, signature_date_signed" のような具体的 error message を詳細に出すようにしてください。Claude が問題を修正しやすくなります。

### Package dependencies

Skills はプラットフォーム固有の制約を持つ code execution environment で動きます:

* **claude.ai**: npm と PyPI から package を install でき、GitHub repository から pull できる
* **Anthropic API**: network access がなく、runtime package installation もできない

必要な package は SKILL.md に列挙し、[code execution tool documentation](/en/docs/agents-and-tools/tool-use/code-execution-tool) で利用可能か確認してください。

### Runtime environment

Skills は filesystem access、bash command、code execution capabilities を持つ code execution environment で動きます。この architecture の概念説明は、overview の [The Skills architecture](/en/docs/agents-and-tools/agent-skills/overview#the-skills-architecture) を参照してください。

**これが authoring に与える影響:**

**Claude が Skills にアクセスする方法:**

1. **Metadata は事前ロード**: 起動時に、すべての Skills の YAML frontmatter から name と description が system prompt にロードされる
2. **ファイルはオンデマンドで読む**: Claude は必要なときに bash Read tools を使って filesystem から SKILL.md や他ファイルにアクセスする
3. **script は効率よく実行される**: utility script は全文を context に読み込まず bash で実行できる。token を消費するのは output だけ
4. **大きな file に context penalty はない**: reference file、data、documentation は実際に読まれるまで context token を消費しない

* **file path は重要**: Claude は skill directory を filesystem のように辿る。backslash ではなく forward slash（`reference/guide.md`）を使う
* **ファイル名は説明的に**: `doc2.md` ではなく `form_validation_rules.md` のように内容が分かる名前を使う
* **discovery のために整理する**: directory は domain または feature ごとに構成する
  * Good: `reference/finance.md`, `reference/sales.md`
  * Bad: `docs/file1.md`, `docs/file2.md`
* **包括的な resource を束ねる**: 完全な API docs、広範な例、大きな dataset を含めてもよい。アクセスされるまで context penalty はない
* **決定的な操作には script を優先する**: Claude に validation code を生成させるより `validate_form.py` を書く
* **execution intent を明確にする**:
  * "Run `analyze_form.py` to extract fields"（実行）
  * "See `analyze_form.py` for the extraction algorithm"（reference として読む）
* **file access pattern をテストする**: 実際の request で Claude が directory structure を辿れるか確認する

**例:**

```
bigquery-skill/
├── SKILL.md (overview, points to reference files)
└── reference/
    ├── finance.md (revenue metrics)
    ├── sales.md (pipeline data)
    └── product.md (usage analytics)
```

ユーザーが revenue を尋ねると、Claude は SKILL.md を読み、`reference/finance.md` への reference を見て、その file だけを bash で読みます。sales.md や product.md は必要になるまで filesystem 上に残り、context token を 0 しか消費しません。この filesystem-based model が progressive disclosure を可能にします。Claude は task ごとに必要なものだけを選択的に読み込めます。

技術 architecture の完全な詳細は、Skills overview の [How Skills work](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work) を参照してください。

### MCP tool references

Skill が MCP（Model Context Protocol）tools を使う場合は、"tool not found" エラーを避けるため、常に完全修飾の tool 名を使ってください。

**形式**: `ServerName:tool_name`

**例:**

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

ここで:

* `BigQuery` と `GitHub` は MCP server 名
* `bigquery_schema` と `create_issue` はその server 内の tool 名

server prefix がないと、特に複数の MCP server がある場合に、Claude が tool を見つけられないことがあります。

### tool が入っている前提を置かない

package が利用可能だと決めつけないでください:

````markdown  theme={null}
**Bad example: Assumes installation**:
"Use the pdf library to process the file."

**Good example: Explicit about dependencies**:
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## Technical notes

### YAML frontmatter requirements

SKILL.md の frontmatter には `name`（最大 64 文字）と `description`（最大 1024 文字）が必要です。完全な構造の詳細は [Skills overview](/en/docs/agents-and-tools/agent-skills/overview#skill-structure) を参照してください。

### Token budgets

最適な性能のため、SKILL.md 本文は 500 行未満に保ってください。これを超えるなら、前述の progressive disclosure pattern を使って別ファイルへ分けます。architecture の詳細は [Skills overview](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work) を参照してください。

## Checklist for effective Skills

Skill を共有する前に次を確認してください:

### Core quality

* [ ] Description が具体的で key terms を含んでいる
* [ ] Description に Skill が何をするかと、いつ使うかの両方が入っている
* [ ] SKILL.md 本文が 500 行未満
* [ ] 追加詳細が別ファイルにある（必要な場合）
* [ ] 時間依存情報がない（または "old patterns" section にある）
* [ ] 用語が一貫している
* [ ] 例が抽象的ではなく具体的
* [ ] file references が 1 階層だけ
* [ ] progressive disclosure が適切に使われている
* [ ] workflow の step が明確

### Code and scripts

* [ ] script が問題を解決しており、Claude に丸投げしていない
* [ ] error handling が明示的で役立つ
* [ ] "voodoo constants" がない（すべての値に根拠がある）
* [ ] 必要 package が指示に列挙され、利用可能と確認済み
* [ ] script の documentation が明確
* [ ] Windows-style path がない（すべて forward slashes）
* [ ] 重要操作に validation/verification step がある
* [ ] 品質が重要な task に feedback loop が含まれている

### Testing

* [ ] 少なくとも 3 つの evaluation を作成した
* [ ] Haiku、Sonnet、Opus でテストした
* [ ] 実利用シナリオでテストした
* [ ] チームのフィードバックを取り込んだ（該当する場合）

## Next steps

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="/en/docs/agents-and-tools/agent-skills/quickstart">
    最初の Skill を作る
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="/en/docs/claude-code/skills">
    Claude Code で Skills を作成・管理する
  </Card>

  <Card title="Use Skills with the API" icon="code" href="/en/api/skills-guide">
    Skills をアップロードし、プログラムから使う
  </Card>
</CardGroup>

