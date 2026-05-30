# Code Quality Reviewer プロンプトテンプレート

コード品質 reviewer サブエージェントを dispatch するときにこのテンプレートを使います。

**目的:** 実装が適切に構築されていることを確認する（クリーンさ、テスト、保守性）

**仕様準拠レビューが通ってからのみ dispatch してください。**

```
Task tool (general-purpose):
  Use template at requesting-code-review/code-reviewer.md

  DESCRIPTION: [task summary, from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
```

**標準的なコード品質の観点に加えて、reviewer は次も確認するべきです:**
- 各ファイルが、明確に定義されたインターフェースを持つ 1 つの責務だけを持っているか？
- 単位が独立して理解・テストできるように分解されているか？
- 実装が計画にあるファイル構成に従っているか？
- この実装で新規ファイルがすでに大きくなっていないか、あるいは既存ファイルを大きく増やしていないか？（既存のファイルサイズ自体は問題にせず、この変更が増やした分に注目すること。）

**Code reviewer の返却内容:** Strengths、Issues（Critical/Important/Minor）、Assessment

