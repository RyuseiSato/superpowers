---
name: verification-before-completion
description: 作業が完了・修正済み・成功していると主張しようとするとき、commit や PR 作成の前に使う。成功を主張する前に検証コマンドの実行と出力確認を必須にする。常に主張より証拠を優先する
---

# Verification Before Completion

## 概要

検証せずに作業完了を主張するのは、効率ではなく不誠実です。

**中核原則:** 常に、主張の前に証拠。

**このルールの文言に反することは、このルールの精神にも反します。**

## 鉄則

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

このメッセージで検証コマンドを実行していないなら、通ったと主張してはいけません。

## ゲート関数

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## よくある失敗

| 主張 | 必要なもの | 不十分なもの |
|-------|----------|----------------|
| テストが通る | テストコマンド出力: failures 0 | 以前の実行結果、「通るはず」 |
| Linter がクリーン | linter 出力: errors 0 | 部分確認、推測 |
| Build 成功 | build コマンド: exit 0 | linter 成功、ログがよさそう |
| バグ修正済み | 元の症状を再現するテストが通る | コード変更済み、直ったはず |
| Regression test が動く | red-green cycle を確認済み | 一度だけテスト通過 |
| Agent が完了した | VCS diff で変更が見える | Agent の "success" 報告 |
| 要件を満たした | 行ごとのチェックリスト | テスト通過 |

## レッドフラグ - STOP

- "should"、"probably"、"seems to" を使っている
- 検証前に満足を表明している（"Great!"、"Perfect!"、"Done!" など）
- 検証せずに commit / push / PR をしようとしている
- agent の success report を信じている
- 部分的な検証に依存している
- 「今回だけ」と考えている
- 疲れて早く終わらせたくなっている
- **検証を走らせていないのに成功を示唆するあらゆる表現**

## 合理化の防止

| 言い訳 | 現実 |
|--------|---------|
| "Should work now" | 検証を実行する |
| "I'm confident" | 自信 ≠ 証拠 |
| "Just this once" | 例外はない |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | 独立に検証する |
| "I'm tired" | 疲労は言い訳にならない |
| "Partial check is enough" | 部分確認では何も証明できない |
| "Different words so rule doesn't apply" | 文面より精神 |

## 重要パターン

**テスト:**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**Regression tests（TDD の Red-Green）:**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**Build:**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**要件:**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**Agent への委譲:**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## これが重要な理由

24 件の failure memory から:
- 人間のパートナーに "I don't believe you" と言われた - 信頼が壊れた
- 未定義関数が出荷されるところだった - クラッシュする
- 要件不足のまま出荷されるところだった - 機能不完全
- 虚偽の完了報告で時間を浪費 → 軌道修正 → 手戻り
- 次の原則に反する: "Honesty is a core value. If you lie, you'll be replaced."

## 適用するタイミング

**必ず適用する場面:**
- 成功・完了を示すすべての表現の前
- 満足を示すすべての表現の前
- 作業状態についてのあらゆるポジティブな発言の前
- commit、PR 作成、タスク完了の前
- 次のタスクへ進む前
- agent へ委譲する前

**このルールが適用されるもの:**
- 完全一致の文言
- 言い換えや同義表現
- 成功を示唆する含意
- 完了・正しさを示すあらゆるコミュニケーション

## 要点

**検証に近道はありません。**

コマンドを実行する。出力を読む。その後で結果を主張する。

これは交渉不可です。

