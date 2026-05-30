# Creation Log: Systematic Debugging Skill

重要な skill を抽出し、構造化し、堅牢化する際の参考例。

## ソース資料

`~/.claude/CLAUDE.md` からデバッグ framework を抽出:
- 4-phase の systematic process（Investigation → Pattern Analysis → Hypothesis → Implementation）
- 中核命令: root cause は常に見つけ、症状は決して直さない
- 時間的プレッシャーや合理化に耐えるよう設計されたルール

## 抽出時の判断

**含めるもの:**
- すべてのルールを含む完全な 4-phase framework
- 近道を防ぐ指示（"NEVER fix symptom"、"STOP and re-analyze"）
- プレッシャー耐性のある表現（"even if faster"、"even if I seem in a hurry"）
- 各 phase の具体的な手順

**除外するもの:**
- プロジェクト固有の文脈
- 同じルールの反復的な言い換え
- 物語的な説明（原則へ圧縮）

## `skill-creation/SKILL.md` に従った構成

1. **豊富な when_to_use** - 症状とアンチパターンを含めた
2. **Type: technique** - 具体的な手順を伴うプロセス
3. **Keywords** - "root cause", "symptom", "workaround", "debugging", "investigation"
4. **Flowchart** - 「fix failed」から再分析か、さらに fixes を足すかの分岐
5. **Phase ごとの分解** - スキャンしやすい checklist 形式
6. **Anti-patterns section** - やってはいけないこと（この skill では特に重要）

## 堅牢化の要素

この framework は、プレッシャー下の合理化に耐えるよう設計されている:

### 言葉の選び方
- "ALWAYS" / "NEVER"（"should" / "try to" ではない）
- "even if faster" / "even if I seem in a hurry"
- "STOP and re-analyze"（明示的に立ち止まらせる）
- "Don't skip past"（実際に起こりがちな振る舞いを捉える）

### 構造的な防御
- **Phase 1 required** - 実装へ飛ばせない
- **Single hypothesis rule** - 思考を強制し、shotgun fixes を防ぐ
- **Explicit failure mode** - "IF your first fix doesn't work" と、そのときの必須アクション
- **Anti-patterns section** - 近道がどのように見えるかを正確に示す

### 冗長性
- root cause 命令を overview + when_to_use + Phase 1 + implementation rules に重ねて配置
- "NEVER fix symptom" は異なる文脈で 4 回出てくる
- 各 phase に明示的な「飛ばすな」ガイダンスがある

## テスト方針

`skills/meta/testing-skills-with-subagents` に従って 4 つの validation tests を作成:

### Test 1: Academic Context（プレッシャーなし）
- 単純なバグ、時間的プレッシャーなし
- **結果:** 完璧に準拠し、調査も完全

### Test 2: 時間的プレッシャー + 明白な Quick Fix
- ユーザーが "in a hurry"、症状修正が簡単に見える
- **結果:** 近道を拒否し、完全なプロセスに従って本当の root cause を見つけた

### Test 3: 複雑なシステム + 不確実性
- 多層障害で、root cause が見つかるか不明
- **結果:** systematic に調査し、全層を辿って source を発見

### Test 4: 最初の修正が失敗
- hypothesis が外れ、さらに fixes を足したくなる状況
- **結果:** 立ち止まって再分析し、新しい hypothesis を立てた（shotgun にならなかった）

**すべてのテストに合格。** 合理化は見つからなかった。

## 反復

### 初版
- 完全な 4-phase framework
- Anti-patterns section
- 「fix failed」時の判断を示す flowchart

### Enhancement 1: TDD への参照
- `skills/testing/test-driven-development` へのリンクを追加
- TDD の "simplest code" と debugging の "root cause" は異なる、という説明を追加
- 手法の混同を防ぐ

## 最終結果

次を満たす bulletproof な skill:
- ✅ root cause investigation を明確に義務付ける
- ✅ 時間的プレッシャーによる合理化に耐える
- ✅ 各 phase の具体的手順を示す
- ✅ anti-patterns を明示的に示す
- ✅ 複数のプレッシャー状況でテスト済み
- ✅ TDD との関係を明確化
- ✅ 利用準備完了

## 重要な洞察

**最も重要な堅牢化要素:** その瞬間には正当化できそうに見える近道を、Anti-patterns section で正確に示していること。Claude が "I'll just add this one quick fix" と考えたときに、そのまさに同じパターンが誤りとして列挙されているのを見ることで認知的摩擦が生まれる。

## 使用例

バグに遭遇したとき:
1. skill を読み込む: `skills/debugging/systematic-debugging`
2. overview を読む（10 秒）- 命令を思い出す
3. Phase 1 の checklist に従う - 調査が強制される
4. 飛ばしたくなったら - anti-pattern を見て止まる
5. 全 Phase を完了する - root cause を発見

**投資時間:** 5〜10 分
**節約できる時間:** 症状もぐら叩きの何時間分も

---

*Created: 2025-10-03*
*Purpose: skill extraction and bulletproofing の参考例*
