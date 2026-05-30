# ドキュメントレビューシステム設計

## 概要

superpowers のワークフローに、次の 2 つの新しいレビュー段階を追加します。

1. **仕様ドキュメントレビュー** - brainstorming の後、writing-plans の前
2. **計画ドキュメントレビュー** - writing-plans の後、実装の前

どちらも、実装レビューで使われている反復ループパターンに従います。

## 仕様ドキュメントレビュアー

**目的:** 仕様が完全で、一貫しており、実装計画作成の準備が整っていることを確認する。

**場所:** `skills/brainstorming/spec-document-reviewer-prompt.md`

**確認する内容:**

| カテゴリ | 確認内容 |
|----------|----------|
| 完全性 | TODO、プレースホルダー、"TBD"、未完成のセクション |
| カバレッジ | 欠けているエラーハンドリング、エッジケース、統合ポイント |
| 一貫性 | 内部矛盾、競合する要件 |
| 明確さ | 曖昧な要件 |
| YAGNI | 要求されていない機能、過剰設計 |

**出力形式:**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**レビューループ:** 問題が見つかる -> brainstorming agent が修正 -> 再レビュー -> 承認されるまで繰り返す。

**ディスパッチ機構:** `subagent_type: general-purpose` を指定して Task tool を使います。レビュアー用 prompt template が完全な prompt を提供します。brainstorming skill の controller がレビュアーをディスパッチします。

## 計画ドキュメントレビュアー

**目的:** 計画が完全で、仕様と一致しており、適切にタスク分解されていることを確認する。

**場所:** `skills/writing-plans/plan-document-reviewer-prompt.md`

**確認する内容:**

| カテゴリ | 確認内容 |
|----------|----------|
| 完全性 | TODO、プレースホルダー、未完成のタスク |
| 仕様との整合 | 計画が仕様要件をカバーしていること、scope creep がないこと |
| タスク分解 | タスクが原子的で、境界が明確であること |
| タスク構文 | タスクおよびステップの checkbox 構文 |
| チャンクサイズ | 各チャンクが 1000 行未満 |

**チャンクの定義:** チャンクとは、計画ドキュメント内のタスクの論理的なまとまりであり、`## Chunk N: <name>` 見出しで区切られます。writing-plans skill は、論理的なフェーズ（例: "Foundation", "Core Features", "Integration"）に基づいてこれらの境界を作成します。各チャンクは、個別にレビューできるだけの自己完結性を持っている必要があります。

**仕様整合性の検証:** レビュアーは以下の両方を受け取ります。
1. 計画ドキュメント（または現在のチャンク）
2. 参照用の仕様ドキュメントのパス

レビュアーは両方を読み、要件のカバレッジを比較します。

**出力形式:** 仕様レビュアーと同じですが、現在のチャンクにスコープを限定します。

**レビュー手順（チャンク単位）:**
1. Writing-plans がチャンク N を作成する
2. Controller がチャンク N の内容と spec path を付けて plan-document-reviewer をディスパッチする
3. レビュアーがチャンクと仕様を読み、判定を返す
4. 問題がある場合: writing-plans agent がチャンク N を修正し、手順 2 に戻る
5. 承認された場合: チャンク N+1 へ進む
6. すべてのチャンクが承認されるまで繰り返す

**ディスパッチ機構:** 仕様レビュアーと同じく、`subagent_type: general-purpose` を指定した Task tool を使います。

## 更新後のワークフロー

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**Spec Review Loop:**
1. 仕様が完成
2. レビュアーをディスパッチ
3. 問題がある場合: 修正 -> 2 に戻る
4. 承認された場合: 次へ進む

**Plan Review Loop:**
1. チャンク N が完成
2. チャンク N 用レビュアーをディスパッチ
3. 問題がある場合: 修正 -> 2 に戻る
4. 承認された場合: 次のチャンクまたは実装へ

## Markdown タスク構文

タスクとステップには checkbox 構文を使います。

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## エラーハンドリング

**レビューループの終了条件:**
- 反復回数にハードな上限はありません。レビュアーが承認するまでループを続けます。
- ループが 5 回を超えた場合、controller は human にガイダンスを求めるためにこれを通知する必要があります。
- human は次を選べます: 反復を続ける、既知の問題を抱えたまま承認する、中止する

**意見の不一致への対処:**
- レビュアーは助言的存在であり、問題を指摘しますがブロックはしません
- agent がレビュアーのフィードバックが誤りだと考える場合は、自分の修正内でその理由を説明するべきです
- 同じ問題について 3 回反復しても意見の不一致が解消しない場合は、human に通知します

**不正なレビュアー出力:**
- Controller は、レビュアー出力に必要な項目（Status、必要なら Issues）があることを検証する必要があります
- 形式が不正なら、期待される形式について注記を付けてレビュアーを再ディスパッチします
- 不正な応答が 2 回続いたら、human に通知します

## 変更するファイル

**新規ファイル:**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**変更するファイル:**
- `skills/brainstorming/SKILL.md` - spec 書き込み後に review loop を追加
- `skills/writing-plans/SKILL.md` - チャンク単位の review loop を追加し、task syntax の例を更新
