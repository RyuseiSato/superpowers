---
name: executing-plans
description: separate session で、review checkpoint 付きの書かれた implementation plan を実行するときに使う
---

# プランを実行する

## 概要

plan を読み込み、批判的にレビューし、すべての task を実行し、完了時に報告する。

**開始時に宣言する:** "I'm using the executing-plans skill to implement this plan."

**注意:** Superpowers は subagent へのアクセスがあると格段にうまく機能することを `human partner` に伝えること。subagent support のあるプラットフォーム（Claude Code や Codex など）で実行したほうが、作業品質は大幅に高くなる。subagent が利用可能なら、この skill ではなく superpowers:subagent-driven-development を使う。

## プロセス

### Step 1: plan を読み込み、レビューする
1. plan file を読む
2. 批判的にレビューし、plan に関する疑問点や懸念点を洗い出す
3. 懸念がある場合: 開始前に `human partner` に伝える
4. 懸念がない場合: TodoWrite を作成して進む

### Step 2: task を実行する

各 task について:
1. in_progress にする
2. 各 step に正確に従う（plan には小さな step がある）
3. 指定どおりに検証を実行する
4. completed にする

### Step 3: 開発を完了する

すべての task が完了し、検証も終わったら:
- 次を宣言する: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** superpowers:finishing-a-development-branch を使う
- その skill に従って tests を確認し、選択肢を提示し、選ばれた内容を実行する

## 止まって助けを求めるタイミング

**次の場合はただちに実行を止める:**
- blocker に当たった（依存関係不足、テスト失敗、指示不明瞭）
- plan に、着手を妨げる重大な欠落がある
- 指示を理解できない
- 検証が何度も失敗する

**推測するより、明確化を求めること。**

## 前の step を見直すタイミング

**次の場合は Review（Step 1）に戻る:**
- あなたのフィードバックを受けて partner が plan を更新した
- 根本的なアプローチを考え直す必要がある

**blocker を無理に突破しないこと** - 止まって質問する。

## 覚えておくこと
- まず批判的に plan をレビューする
- plan の step に正確に従う
- 検証を省略しない
- plan が skill の参照を求めるときは、その skills を参照する
- 詰まったら止まり、推測しない
- ユーザーの明示的な同意なしに main/master branch で実装を始めてはならない

## Integration

**必須の workflow skills:**
- **superpowers:using-git-worktrees** - isolated workspace を確保する（作成するか、既存のものを確認する）
- **superpowers:writing-plans** - この skill が実行する plan を作成する
- **superpowers:finishing-a-development-branch** - すべての開発完了後の仕上げを行う
