---
name: systematic-debugging
description: あらゆるバグ、テスト失敗、予期しない挙動に遭遇したとき、修正案を出す前に使う
---

# Systematic Debugging

## 概要

場当たり的な修正は時間を浪費し、新しいバグを生む。手早いパッチは根本の問題を覆い隠す。

**中核原則:** 修正を試みる前に、必ず root cause を見つけること。症状への修正は失敗である。

**このプロセスの文言に反することは、デバッグの精神に反することでもある。**

## 鉄則

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

Phase 1 を完了していないなら、修正案を出してはならない。

## いつ使うか

あらゆる技術的な問題に使う:
- テスト失敗
- 本番のバグ
- 予期しない挙動
- パフォーマンス問題
- ビルド失敗
- integration の問題

**特に使うべき場面:**
- 時間的プレッシャーがあるとき（緊急時ほど当てずっぽうになりやすい）
- 「とりあえず一発で直せそう」に見えるとき
- すでに複数の修正を試しているとき
- 前の修正が効かなかったとき
- 問題を十分に理解していないとき

**次の場合でも飛ばさない:**
- 問題が単純に見えるとき（単純なバグにも root cause はある）
- 急いでいるとき（焦りは手戻りを保証する）
- manager が「今すぐ直せ」と言っているとき（systematic な方が空回りより速い）

## 4 つの Phase

次の Phase に進む前に、それぞれの Phase を必ず完了しなければならない。

### Phase 1: Root Cause Investigation

**いかなる修正を試みる前にも:**

1. **エラーメッセージを注意深く読む**
   - errors や warnings を読み飛ばさない
   - そこに正解がそのまま書かれていることが多い
   - stack traces を最後まで読む
   - 行番号、ファイルパス、error codes を記録する

2. **一貫して再現する**
   - 確実に再現できるか？
   - 正確な手順は何か？
   - 毎回起きるか？
   - 再現できないなら → 推測せず、データをさらに集める

3. **最近の変更を確認する**
   - 何が変わって、これを引き起こし得るか？
   - Git diff、最近の commits
   - 新しい dependencies、config の変更
   - 環境差異

4. **複数コンポーネントのシステムでは証拠を集める**

   **システムに複数のコンポーネントがあるとき（CI → build → signing、API → service → database）:**

   **修正案を出す前に、診断用 instrumentation を追加する:**
   ```
   For EACH component boundary:
     - Log what data enters component
     - Log what data exits component
     - Verify environment/config propagation
     - Check state at each layer

   Run once to gather evidence showing WHERE it breaks
   THEN analyze evidence to identify failing component
   THEN investigate that specific component
   ```

   **例（多層システム）:**
   ```bash
   # Layer 1: Workflow
   echo "=== Secrets available in workflow: ==="
   echo "IDENTITY: ${IDENTITY:+SET}${IDENTITY:-UNSET}"

   # Layer 2: Build script
   echo "=== Env vars in build script: ==="
   env | grep IDENTITY || echo "IDENTITY not in environment"

   # Layer 3: Signing script
   echo "=== Keychain state: ==="
   security list-keychains
   security find-identity -v

   # Layer 4: Actual signing
   codesign --sign "$IDENTITY" --verbose=4 "$APP"
   ```

   **これで分かること:** どの layer が失敗しているか（secrets → workflow ✓、workflow → build ✗）

5. **データフローを追跡する**

   **error が call stack の深い場所で起きているとき:**

   完全な逆向き tracing の手法については、このディレクトリの `root-cause-tracing.md` を参照すること。

   **簡易版:**
   - bad value はどこで生まれたか？
   - それを bad value で呼んだのは何か？
   - source を見つけるまで上へ遡り続ける
   - 症状ではなく source で直す

### Phase 2: Pattern Analysis

**修正の前にパターンを見つける:**

1. **動いている例を探す**
   - 同じ codebase 内の似た動作をするコードを探す
   - 壊れているものに似ていて、動いているものは何か？

2. **reference と比較する**
   - pattern を実装しているなら、reference implementation を最後まで読む
   - 流し読みしない。1 行残らず読む
   - 適用する前に pattern を完全に理解する

3. **差分を特定する**
   - 動いているものと壊れているものの違いは何か？
   - どんなに小さくても、すべての違いを列挙する
   - 「それは関係ないはず」と決めつけない

4. **依存関係を理解する**
   - 他にどのコンポーネントが必要か？
   - どの settings、config、environment が必要か？
   - どんな前提を置いているか？

### Phase 3: Hypothesis and Testing

**科学的方法:**

1. **単一の hypothesis を立てる**
   - 明確に述べる: 「Y だから、root cause は X だと思う」
   - 書き出す
   - 曖昧ではなく具体的にする

2. **最小限でテストする**
   - hypothesis を検証するための、できる限り最小の変更を行う
   - 一度に変える変数は 1 つだけ
   - 複数箇所を同時に直さない

3. **続ける前に検証する**
   - うまくいったか？ Yes → Phase 4
   - うまくいかなかった？ 新しい hypothesis を立てる
   - その上に修正を積み増してはならない

4. **分からないとき**
   - 「X が分からない」と言う
   - 分かっているふりをしない
   - 助けを求める
   - さらに調べる

### Phase 4: Implementation

**症状ではなく root cause を直す:**

1. **失敗するテストケースを作る**
   - できる限り単純な再現
   - 可能なら自動テスト
   - framework がなければ一回限りのテストスクリプト
   - 修正前に必須
   - 適切な失敗テストを書くには `superpowers:test-driven-development` skill を使う

2. **単一の修正を実装する**
   - 特定した root cause に対処する
   - 一度に 1 つの変更だけ
   - 「ついでに」改善しない
   - 束ねた refactoring をしない

3. **修正を検証する**
   - テストは通るようになったか？
   - 他のテストは壊れていないか？
   - 問題は本当に解決したか？

4. **修正が効かなかったら**
   - STOP
   - 何回修正を試したか数える
   - 3 回未満なら: Phase 1 に戻り、新しい情報で再分析する
   - **3 回以上なら: STOP して architecture を疑う（下の step 5）**
   - architecture について話し合う前に Fix #4 を試してはならない

5. **3 回以上修正に失敗したら: Architecture を疑う**

   **architectural problem を示すパターン:**
   - 修正のたびに、別の場所に新しい shared state / coupling / 問題が現れる
   - 修正の実装に「大規模な refactoring」が必要になる
   - 修正のたびに別の場所で新しい症状が生まれる

   **STOP して前提を問い直す:**
   - この pattern は根本的に健全か？
   - 私たちは「惰性だけでこれを続けて」いないか？
   - 症状を直し続けるより、architecture を refactor すべきではないか？

   **これ以上修正を試す前に your human partner と相談すること**

   これは failed hypothesis ではなく、間違った architecture である。

## 危険信号 - 止まってプロセスに従う

次のように考え始めたら:
- 「今は quick fix だけして、調査は後で」
- 「とりあえず X を変えて動くか見よう」
- 「複数の変更をまとめて入れて、テストを回そう」
- 「テストは飛ばして手で確認しよう」
- 「たぶん X だ。直してみよう」
- 「完全には分からないけど、これは効きそう」
- 「pattern では X だけど、別の形にアレンジしよう」
- 「主な問題はこれです: [調査せずに修正案を列挙する]」
- データフローを追跡する前に解決策を提案している
- **「もう 1 回だけ修正を試す」(すでに 2 回以上試した後)**
- **修正のたびに別の場所で新しい問題が現れる**

**これらはすべて意味することは同じ: STOP。Phase 1 に戻ること。**

**3 回以上修正に失敗したら:** architecture を疑う（Phase 4.5 を参照）

## 間違っているときの your human partner からのシグナル

**次のような軌道修正に注意する:**
- 「それって起きていないのでは？」 - 確認せずに前提を置いた
- 「それで私たちに見えるのは...？」 - 証拠集めを入れるべきだった
- 「推測をやめて」 - 理解せずに修正案を出している
- 「Ultrathink this」 - 症状だけでなく前提も疑うべき
- 「行き詰まってる？」（苛立ちを込めて） - アプローチが機能していない

**これが見えたら:** STOP。Phase 1 に戻ること。

## よくある合理化

| 言い訳 | 現実 |
|--------|---------|
| "問題は単純だから、プロセスはいらない" | 単純な問題にも root cause はある。単純なバグに対してもこのプロセスは速い。 |
| "緊急だから、プロセスに時間をかけられない" | systematic debugging は、当てずっぽうの空回りより**速い**。 |
| "まずこれを試してから調べよう" | 最初の修正がパターンを決める。最初から正しくやること。 |
| "修正が効くと確認してからテストを書く" | テストされていない修正は定着しない。先にテストすれば証明できる。 |
| "複数の修正を一度にやれば時間短縮" | 何が効いたか切り分けられない。新しいバグの原因になる。 |
| "reference は長いから、pattern をアレンジして使おう" | 中途半端な理解は必ずバグを生む。全部読むこと。 |
| "問題は見えた。直そう" | 症状が見えたこと ≠ root cause を理解したこと。 |
| "もう 1 回だけ修正を試す"（2 回以上失敗した後） | 3 回以上の失敗 = architectural problem。もう直そうとせず、pattern を疑う。 |

## クイックリファレンス

| Phase | 主な活動 | 成功条件 |
|-------|---------------|------------------|
| **1. Root Cause** | errors を読む、再現する、変更を確認する、証拠を集める | WHAT と WHY を理解している |
| **2. Pattern** | 動いている例を探す、比較する | 差分を特定できる |
| **3. Hypothesis** | 仮説を立てる、最小限で試す | 確認できた、または新しい hypothesis が必要 |
| **4. Implementation** | テストを作る、修正する、検証する | バグが解消し、テストが通る |

## プロセスの結果、「root cause がない」と分かったとき

systematic な調査の結果、問題が本当に environment 依存、timing 依存、または外部要因だと分かったなら:

1. プロセスは完了している
2. 何を調査したか文書化する
3. 適切な対処を実装する（retry、timeout、error message）
4. 将来の調査のために monitoring / logging を追加する

**ただし:** 「root cause がない」ケースの 95% は、調査不足である。

## 補助テクニック

これらのテクニックは systematic debugging の一部であり、このディレクトリで利用できる:

- **`root-cause-tracing.md`** - call stack を逆向きに辿って元のトリガーを見つける
- **`defense-in-depth.md`** - root cause を見つけた後で複数層に validation を追加する
- **`condition-based-waiting.md`** - 任意の timeout を condition polling に置き換える

**関連 skills:**
- **superpowers:test-driven-development** - 失敗するテストケースを作るため（Phase 4, Step 1）
- **superpowers:verification-before-completion** - 成功を主張する前に修正が効いたか検証するため

## 実務上の効果

デバッグセッションから:
- systematic なアプローチ: 修正まで 15〜30 分
- 場当たり的な修正アプローチ: 2〜3 時間の空回り
- 1 回目で直る率: 95% vs 40%
- 新しいバグの混入: ほぼゼロ vs よく起こる
