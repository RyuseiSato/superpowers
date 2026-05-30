---
name: writing-plans
description: spec または複数ステップのタスクに対する要件があり、コードに触る前に使う
---

# プランを書く

## 概要

実装者がこのコードベースについての文脈をまったく持たず、しかも趣味がいまひとつだと仮定して、包括的な実装 plan を書く。その人が知る必要のあることをすべて文書化すること: 各 task でどの files に触るか、確認すべき code・testing・docs、どうテストするか。全体の plan を、小さく実行しやすい task に分けて渡す。DRY。YAGNI。TDD。こまめな commit。

熟練した開発者ではあるが、こちらの toolset や問題領域についてはほとんど何も知らないと想定する。また、良いテスト設計についても十分には分かっていないと想定する。

**開始時に宣言する:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** isolated な worktree で作業している場合、それは実行時に `superpowers:using-git-worktrees` skill によって作成されているはずである。

**plan の保存先:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （plan の保存場所に関するユーザー設定がある場合は、その設定を優先する）

## スコープ確認

spec が複数の独立したサブシステムを扱っている場合、ブレインストーミング時点でサブプロジェクトごとの spec に分解されているべきである。そうなっていないなら、サブシステムごとに plan を分けることを提案する。各 plan は、それ単体で動作し、テスト可能なソフトウェアを生み出すべきである。

## ファイル構成

タスクを定義する前に、どの files を新規作成・変更するのか、その各 file が何を担うのかを整理する。分解の判断が固まるのはこの段階である。

- 境界が明確で、よく定義されたインターフェースを持つ単位として設計する。各 file は一つの明確な責務だけを持つべきである。
- あなたは、一度に文脈に保持できるコードについて最もよく推論でき、files が焦点化されているほど編集の信頼性も高い。やりすぎな大きい files より、小さく焦点の合った files を優先する。
- 一緒に変更される files は近くに置く。技術レイヤーではなく責務で分ける。
- 既存コードベースでは、確立されたパターンに従う。コードベースが大きな files を使うなら、一方的に再構成してはならない。ただし、変更対象の file が扱いにくく肥大化しているなら、plan に分割を含めるのは妥当である。

この構造が task 分解の指針になる。各 task は、それ単体でも意味が通る自己完結した変更を生み出すべきである。

## 小さな task 粒度

**各 step は 1 アクション（2〜5 分）にする:**
- 「失敗するテストを書く」 - step
- 「失敗することを確認するために実行する」 - step
- 「テストを通す最小限のコードを実装する」 - step
- 「テストが通ることを確認するために実行する」 - step
- 「commit する」 - step

## Plan Document Header

**すべての plan は必ずこのヘッダーで始めること:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## プレースホルダー禁止

各 step には、実装者が必要とする実際の内容を必ず含める。以下は **plan failure** であり、絶対に書いてはならない:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above"（実際のテストコードなし）
- "Similar to Task N"（コードを繰り返して書くこと — 実装者は task を順不同で読むかもしれない）
- 何をするかだけを書き、どうやるかを示していない step（code step には code block 必須）
- どの task にも定義されていない型、関数、メソッドへの参照

## 覚えておくこと
- file path は常に正確に
- 各 step の code は完全なものにする — step が code を変更するなら、その code を示す
- command は正確に、期待される出力も書く
- DRY, YAGNI, TDD, こまめな commit

## セルフレビュー

完全な plan を書いたら、spec を新鮮な目で見直し、その spec に対して plan を確認する。これは自分で回すチェックリストであり、subagent の dispatch ではない。

**1. spec coverage:** spec の各 section / requirement に目を通す。どの task がそれを実装するのか示せるか。抜けがあれば列挙する。

**2. placeholder scan:** plan の中に危険信号がないか探す — 上の「プレースホルダー禁止」セクションにあるパターンを確認する。見つけたら修正する。

**3. type consistency:** 後半の task で使った型、メソッドシグネチャ、property 名は、前半の task で定義したものと一致しているか。Task 3 では `clearLayers()` なのに Task 7 では `clearFullLayers()` になっている、というのはバグである。

問題が見つかったら、その場で修正する。再レビューは不要。修正して先へ進めばよい。spec の requirement に対応する task がないと分かったら、その task を追加する。

## 実行への引き継ぎ

plan を保存したら、実行方法の選択肢を提示する:

**"Plan の作成が完了し、`docs/superpowers/plans/<filename>.md` に保存しました。実行方法は 2 つあります:**

**1. Subagent-Driven（推奨）** - task ごとに新しい subagent を dispatch し、task 間で review しながら高速に反復

**2. Inline Execution** - このセッション内で executing-plans を使って task を実行し、checkpoint ごとに review

**どちらのアプローチにしますか？"**

**Subagent-Driven を選んだ場合:**
- **REQUIRED SUB-SKILL:** superpowers:subagent-driven-development を使う
- task ごとに fresh な subagent + 2 段階 review

**Inline Execution を選んだ場合:**
- **REQUIRED SUB-SKILL:** superpowers:executing-plans を使う
- checkpoint を使った batch execution で review する
