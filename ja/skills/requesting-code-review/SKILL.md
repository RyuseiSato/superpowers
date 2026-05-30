---
name: requesting-code-review
description: タスク完了時、主要機能の実装後、または merge 前に、作業が要件を満たしているか確認したいときに使う
---

# Requesting Code Review

問題が連鎖する前に、コード reviewer サブエージェントを dispatch して問題を捕捉します。reviewer には評価に必要な文脈だけを精密に渡し、あなたのセッション履歴は渡しません。これにより reviewer はあなたの思考過程ではなく成果物に集中でき、あなた自身のコンテキストもその後の作業のために温存できます。

**中核原則:** 早くレビューし、頻繁にレビューする。

## レビューを依頼するタイミング

**必須:**
- subagent-driven development では各タスクの後
- 主要機能を完了した後
- main に merge する前

**任意だが価値が高い場面:**
- 行き詰まったとき（新しい視点が得られる）
- リファクタリング前（ベースライン確認）
- 複雑なバグ修正の後

## 依頼方法

**1. git SHA を取得する:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. コード reviewer サブエージェントを dispatch する:**

Task tool の `general-purpose` タイプを使い、`code-reviewer.md` のテンプレートを埋めます。

**プレースホルダー:**
- `{DESCRIPTION}` - 何を作ったかの簡潔な要約
- `{PLAN_OR_REQUIREMENTS}` - 何をするべきだったか
- `{BASE_SHA}` - 開始コミット
- `{HEAD_SHA}` - 終了コミット

**3. フィードバックに対応する:**
- Critical issues は直ちに修正する
- Important issues は次へ進む前に修正する
- Minor issues は後で対応するために記録する
- reviewer が間違っているなら、理由を添えて反論する

## 例

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch code reviewer subagent]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## ワークフローとの統合

**Subagent-Driven Development:**
- 各タスクの後にレビューする
- 問題が積み上がる前に捕まえる
- 次のタスクへ進む前に修正する

**Executing Plans:**
- 各タスク後、または自然なチェックポイントでレビューする
- フィードバックを得て、適用し、続行する

**Ad-Hoc Development:**
- merge 前にレビューする
- 行き詰まったらレビューする

## レッドフラグ

**絶対にしないこと:**
- 「簡単だから」とレビューを飛ばす
- Critical issues を無視する
- 未修正の Important issues があるまま進む
- 妥当な技術的フィードバックに言い争いで応じる

**reviewer が間違っている場合:**
- 技術的根拠を示して反論する
- 動作を証明するコードやテストを示す
- 明確化を依頼する

テンプレート参照先: requesting-code-review/code-reviewer.md

