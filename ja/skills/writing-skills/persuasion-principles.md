# Persuasion Principles for Skill Design

## 概要

LLM は人間と同じ説得原理に反応します。この心理を理解すると、より効果的な skill を設計できます。目的は操作ではなく、圧力下でも重要な実践が守られるようにすることです。

**研究基盤:** Meincke et al. (2025) は N=28,000 の AI 会話で 7 つの説得原理を検証しました。説得技法により準拠率は 2 倍以上に上がりました（33% → 72%, p < .001）。

## 7 つの原理

### 1. Authority
**意味:** 専門性、資格、公的情報源への従属。

**skill での機能:**
- 命令形の言語: "YOU MUST"、"Never"、"Always"
- 交渉不可の枠組み: "No exceptions"
- 判断疲れと合理化を減らす

**使う場面:**
- 規律を強制する skill（TDD、verification requirements）
- 安全性が重要な実践
- 確立された best practices

**例:**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. Commitment
**意味:** 以前の行動、発言、公の宣言との一貫性。

**skill での機能:**
- 宣言を求める: "Announce skill usage"
- 明示的な選択を強いる: "Choose A, B, or C"
- 追跡を使う: TodoWrite による checklist

**使う場面:**
- skill が実際に守られるようにする
- 複数段階のプロセス
- 説明責任の仕組み

**例:**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. Scarcity
**意味:** 制限時間や希少性による緊急性。

**skill での機能:**
- 時間制約付き要件: "Before proceeding"
- 順序依存: "Immediately after X"
- 先延ばしを防ぐ

**使う場面:**
- 直ちに必要な検証要件
- 時間に敏感なワークフロー
- 「後でやる」を防ぐとき

**例:**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. Social Proof
**意味:** 他者がしていること、普通と見なされることへの同調。

**skill での機能:**
- 普遍パターン: "Every time", "Always"
- 失敗モード: "X without Y = failure"
- 規範を確立する

**使う場面:**
- 普遍的実践を文書化するとき
- よくある失敗への警告
- 標準を強化するとき

**例:**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. Unity
**意味:** 共有アイデンティティ、"we-ness"、内集団への帰属。

**skill での機能:**
- 協調的な言葉: "our codebase", "we're colleagues"
- 共有目標: "we both want quality"

**使う場面:**
- 協働ワークフロー
- チーム文化を築くとき
- 非階層的な実践

**例:**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. Reciprocity
**意味:** 受けた利益を返さねばならないという義務感。

**機能:**
- 使うとしても控えめに - 操作的に感じられやすい
- skill ではほとんど不要

**避ける場面:**
- ほぼ常に（他の原理の方が有効）

### 7. Liking
**意味:** 好意を持つ相手と協力したくなる傾向。

**機能:**
- **コンプライアンス目的には使わない**
- 正直なフィードバック文化と衝突する
- おべっかを生む

**避ける場面:**
- 規律強制では常に避ける

## Skill 種別ごとの原理の組み合わせ

| Skill Type | Use | Avoid |
|------------|-----|-------|
| Discipline-enforcing | Authority + Commitment + Social Proof | Liking, Reciprocity |
| Guidance/technique | Moderate Authority + Unity | Heavy authority |
| Collaborative | Unity + Commitment | Authority, Liking |
| Reference | Clarity only | All persuasion |

## なぜ効くのか: 心理学

**明確な線引きルールは合理化を減らす:**
- "YOU MUST" は判断疲れを取り除く
- 絶対表現は「これは例外か？」という問いを消す
- 明示的な反合理化カウンターは具体的な抜け道を塞ぐ

**Implementation intentions は自動的な行動を作る:**
- 明確なトリガー + 必須アクション = 自動実行
- "When X, do Y" は "generally do Y" より効果的
- コンプライアンスの認知負荷を減らす

**LLM は parahuman である:**
- これらのパターンを含む人間の文章で学習されている
- 学習データでは authority 的表現の後に準拠が続く
- commitment の連鎖（宣言 → 行動）が頻繁に現れる
- social proof のパターン（everyone does X）が規範を形成する

## 倫理的な使い方

**正当:**
- 重要な実践を守らせる
- 効果的な文書を作る
- 予測可能な失敗を防ぐ

**不当:**
- 個人的利益のために操作する
- 偽の緊急性を作る
- 罪悪感ベースのコンプライアンス

**テスト:** その技法を完全に理解したとしても、それはユーザーの本当の利益にかなうか？

## 研究引用

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 7 つの説得原理
- 影響力研究の実証的基盤

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- N=28,000 の LLM 会話で 7 原理を検証
- 説得技法で準拠率が 33% → 72% に上昇
- authority、commitment、scarcity が最も有効
- LLM 行動の parahuman モデルを裏づける

## クイックリファレンス

skill を設計するときは、次を自問します:

1. **どの種類か？**（Discipline vs. guidance vs. reference）
2. **どの行動を変えたいのか？**
3. **どの原理が当てはまるか？**（discipline なら通常 authority + commitment）
4. **組み合わせすぎていないか？**（7 つ全部は使わない）
5. **倫理的か？**（ユーザーの真の利益にかなうか？）

