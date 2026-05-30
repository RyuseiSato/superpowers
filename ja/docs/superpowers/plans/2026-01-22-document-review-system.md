# ドキュメントレビューシステム実装計画

> **エージェントワーカー向け:** 必須: この計画の実装には superpowers:subagent-driven-development（サブエージェントが利用可能な場合）または superpowers:executing-plans を使用すること。

**目標:** brainstorming スキルと writing-plans スキルに、spec と plan ドキュメントのレビューループを追加する。

**アーキテクチャ:** 各スキルディレクトリに reviewer prompt template を作成する。ドキュメント作成後にレビューループを追加するようスキルファイルを修正する。reviewer のディスパッチには general-purpose subagent と Task tool を使う。

**技術スタック:** Markdown スキルファイル、Task tool 経由の subagent dispatch

**仕様:** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## Chunk 1: Spec ドキュメントレビュアー

この chunk では、brainstorming スキルに spec ドキュメントレビュアーを追加する。

### Task 1: Spec ドキュメントレビュアー Prompt Template を作成する

**Files:**
- Create: `skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **Step 1:** reviewer prompt template ファイルを作成する

```markdown
# Spec Document Reviewer Prompt Template

Use this template when dispatching a spec document reviewer subagent.

**Purpose:** Verify the spec is complete, consistent, and ready for implementation planning.

**Dispatch after:** Spec document is written to docs/superpowers/specs/

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
```

- [ ] **Step 2:** ファイルが正しく作成されたことを確認する

Run: `cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
Expected: ヘッダーと purpose セクションが表示される

- [ ] **Step 3:** Commit

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### Task 2: Brainstorming Skill にレビューループを追加する

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

- [ ] **Step 1:** 現在の brainstorming skill を読む

Run: `cat skills/brainstorming/SKILL.md`

- [ ] **Step 2:** "After the Design" の後に review loop セクションを追加する

"After the Design" セクションを見つけ、documentation の後・implementation の前に新しい "Spec Review Loop" セクションを追加する:

```markdown
**Spec Review Loop:**
After writing the spec document:
1. Dispatch spec-document-reviewer subagent (see spec-document-reviewer-prompt.md)
2. If ❌ Issues Found:
   - Fix the issues in the spec document
   - Re-dispatch reviewer
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to implementation setup

**Review loop guidance:**
- Same agent that wrote the spec fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect
```

- [ ] **Step 3:** 変更を確認する

Run: `grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
Expected: 新しい review loop セクションが表示される

- [ ] **Step 4:** Commit

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## Chunk 2: Plan ドキュメントレビュアー

この chunk では、writing-plans スキルに plan ドキュメントレビュアーを追加する。

### Task 3: Plan ドキュメントレビュアー Prompt Template を作成する

**Files:**
- Create: `skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **Step 1:** reviewer prompt template ファイルを作成する

```markdown
# Plan Document Reviewer Prompt Template

Use this template when dispatching a plan document reviewer subagent.

**Purpose:** Verify the plan chunk is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** Each plan chunk is written

```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
```

- [ ] **Step 2:** ファイルが作成されたことを確認する

Run: `cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
Expected: ヘッダーと purpose セクションが表示される

- [ ] **Step 3:** Commit

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### Task 4: Writing-Plans Skill にレビューループを追加する

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 現在の skill ファイルを読む

Run: `cat skills/writing-plans/SKILL.md`

- [ ] **Step 2:** chunk ごとの review セクションを追加する

"Execution Handoff" セクションの前に追加する:

```markdown
## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.
```

- [ ] **Step 3:** task syntax の例を checkbox を使う形に更新する

Task Structure セクションを変更して checkbox syntax を示す:

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **Step 4:** review loop セクションが追加されたことを確認する

Run: `grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
Expected: 新しい review loop セクションが表示される

- [ ] **Step 5:** task syntax の例が更新されたことを確認する

Run: `grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
Expected: checkbox syntax `### Task N:` が表示される

- [ ] **Step 6:** Commit

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## Chunk 3: Plan ドキュメントヘッダーの更新

この chunk では、新しい checkbox syntax 要件を参照するように plan ドキュメントのヘッダーテンプレートを更新する。

### Task 5: Writing-Plans Skill の Plan Header Template を更新する

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 現在の plan header template を読む

Run: `grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **Step 2:** checkbox syntax を参照するように header template を更新する

plan header では、tasks と steps が checkbox syntax を使うことを明記する。header comment を次のように更新する:

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **Step 3:** 変更を確認する

Run: `grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
Expected: checkbox syntax への言及を含む更新済みヘッダーが表示される

- [ ] **Step 4:** Commit

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```

