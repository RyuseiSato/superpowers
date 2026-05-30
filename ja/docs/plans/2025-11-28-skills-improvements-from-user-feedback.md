# ユーザーフィードバックによるスキル改善

**Date:** 2025-11-28
**Status:** Draft
**Source:** 実際の開発シナリオで superpowers を使った 2 つの Claude インスタンス

---

## エグゼクティブサマリー

2 つの Claude インスタンスが、実際の開発セッションに基づく詳細なフィードバックを提供した。このフィードバックは、現在のスキルに**体系的な欠落**があり、そのスキルに従っていても防げたはずのバグが出荷されてしまったことを示している。

**重要な洞察:** これは単なる解決策の提案ではなく、問題報告である。問題は現実に存在し、解決策は慎重に評価する必要がある。

**主要テーマ:**
1. **検証の抜け** - 操作が成功したことは確認しているが、意図した結果が得られたことまでは確認していない
2. **プロセス衛生** - バックグラウンドプロセスが蓄積し、subagent 間で干渉する
3. **コンテキスト最適化** - subagent に無関係な情報を渡しすぎている
4. **自己振り返りの欠如** - 引き継ぎ前に自分の作業を批評する促しがない
5. **モックの安全性** - モックがインターフェースから乖離しても検出されないことがある
6. **スキルの発動** - スキルは存在するが、読まれず使われていない

---

## 特定された問題

### 問題 1: 設定変更の検証ギャップ

**何が起きたか:**
- Subagent が「OpenAI integration」をテストした
- `OPENAI_API_KEY` env var を設定した
- status 200 のレスポンスを得た
- 「OpenAI integration working」と報告した
- **しかし** レスポンスには `"model": "claude-sonnet-4-20250514"` が含まれていた - 実際には Anthropic を使っていた

**根本原因:**
`verification-before-completion` は操作が成功したかは確認するが、結果が意図した設定変更を反映しているかまでは確認しない。

**影響:** 高 - 統合テストへの誤った安心感が生まれ、バグが本番に出荷される

**典型的な失敗パターン:**
- LLM provider を切り替える → status 200 は確認するが model name は確認しない
- feature flag を有効化する → エラーがないことは確認するが feature が有効になったかは確認しない
- environment を変更する → deployment 成功は確認するが environment vars は確認しない

---

### 問題 2: バックグラウンドプロセスの蓄積

**何が起きたか:**
- セッション中に複数の subagent が dispatch された
- それぞれがバックグラウンドの server process を起動した
- プロセスが蓄積した（4+ servers running）
- 古いプロセスが引き続き port を占有していた
- 後続の E2E test が誤った設定の古い server に当たった
- 混乱を招く / 不正確な test result になった

**根本原因:**
Subagent は stateless であり、以前の subagent が起動したプロセスを知らない。cleanup protocol もない。

**影響:** 中〜高 - テストが誤った server に当たる、誤った pass / failure、デバッグの混乱

---

### 問題 3: Subagent Prompt の Context Bloat

**何が起きたか:**
- 標準的なやり方: subagent に plan file 全体を読ませる
- 実験: task + pattern + file + verify command だけを渡す
- 結果: より速く、より集中でき、1 回で完了することが増えた

**根本原因:**
Subagent が無関係な plan section に token と注意力を浪費している。

**影響:** 中 - 実行が遅くなり、失敗する試行が増える

**うまくいったもの:**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 問題 4: 引き継ぎ前の自己振り返りがない

**何が起きたか:**
- 自己振り返りの prompt を追加した: "Look at your work with fresh eyes - what could be better?"
- Task 5 の implementer は、失敗している test の原因が test bug ではなく implementation bug だと特定した
- line 99 の `strings.Join(metadata.Entrypoint, " ")` が不正な Docker syntax を作っていることを突き止めた
- 自己振り返りがなければ、根本原因を示さずに「test fails」とだけ報告していたはず

**根本原因:**
Implementer は、完了を報告する前に一歩引いて自分の作業を批評することを自然には行わない。

**影響:** 中 - implementer 自身が見つけられたはずのバグが reviewer に渡されてしまう

---

### 問題 5: Mock と Interface の乖離

**何が起きたか:**
```typescript
// Interface defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) defines cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // Wrong!
  })),
}));
```
- Tests passed
- Runtime crashed: "adapter.cleanup is not a function"

**根本原因:**
Mock が bug のある code の呼び出しを元に作られており、interface definition を元にしていなかった。TypeScript は、誤った method name を持つ inline mock を検出できない。

**影響:** 高 - テストが誤った安心感を与え、runtime で crash する

**なぜ testing-anti-patterns では防げなかったか:**
この skill は mock behavior のテストや理解のない mocking は扱っているが、「implementation ではなく interface から mock を導く」という具体的なパターンまでは扱っていない。

---

### 問題 6: Code Reviewer の File Access

**何が起きたか:**
- Code reviewer subagent が dispatch された
- test file を見つけられなかった: "The file doesn't appear to exist in the repository"
- file は実際には存在していた
- reviewer は最初に明示的に読む必要があると分かっていなかった

**根本原因:**
Reviewer prompt に、file を明示的に読む指示が含まれていない。

**影響:** 低〜中 - review が失敗する、または不完全になる

---

### 問題 7: Fix Workflow の遅延

**何が起きたか:**
- Implementer が自己振り返りの中で bug を特定する
- Implementer は fix 方法も分かっている
- 現在の workflow: report → 自分が fixer を dispatch → fixer が修正 → 自分が verify
- 余分な往復が発生し、価値を増やさないまま latency だけが増える

**根本原因:**
Implementer がすでに診断できている場合でも、implementer と fixer の役割が硬直的に分離されている。

**影響:** 低 - latency は増えるが、正しさの問題ではない

---

### 問題 8: スキルが読まれていない

**何が起きたか:**
- `testing-anti-patterns` skill は存在する
- 人間も subagent も、tests を書く前にそれを読まなかった
- いくつかの問題は防げたはずだった（ただし全部ではない - 問題 5 を参照）

**根本原因:**
Subagent に関連する skill を読むことを強制していない。どの prompt にも skill を読む指示が含まれていない。

**影響:** 中 - 使われなければ、skill への投資が無駄になる

---

## 提案される改善

### 1. verification-before-completion: 設定変更の検証を追加する

**追加する新セクション:**

```markdown
## Verifying Configuration Changes

設定、provider、feature flag、environment の変更をテストするとき:

**操作が成功したことだけを確認してはいけません。出力が意図した変更を反映していることを確認してください。**

### Common Failure Pattern

何らかの有効な config が存在するため操作自体は成功するが、テストしたかった config ではない。

### Examples

| 変更 | 不十分 | 必須 |
|--------|-------------|----------|
| LLM provider を切り替える | Status 200 | レスポンスに期待する model name が含まれる |
| feature flag を有効化する | エラーなし | feature behavior が実際に有効 |
| environment を変更する | Deploy 成功 | logs / vars が新しい environment を参照している |
| credentials を設定する | Auth 成功 | 認証された user / context が正しい |

### Gate Function

```
設定変更が動作すると主張する BEFORE:

1. IDENTIFY: この変更後に何が DIFFERENT になっているべきか？
2. LOCATE: その差分はどこで観測できるか？
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: その観測可能な差分を示す command
4. VERIFY: 出力に期待した差分が含まれている
5. ONLY THEN: 設定変更が動作すると主張する

Red flags:
  - 内容を確認せずに "Request succeeded" と言う
  - response body を見ずに status code だけ確認する
  - ポジティブな確認なしにエラーがないことだけを確認する
```

**これが有効な理由:**
単に操作成功を確認するのではなく、INTENT が満たされたことの検証を強制するため。

---

### 2. subagent-driven-development: E2E Tests のための Process Hygiene を追加する

**追加する新セクション:**

```markdown
## Process Hygiene for E2E Tests

service（servers、databases、message queues）を起動する subagent を dispatch するとき:

### Problem

Subagent は stateless であり、以前の subagent が起動したプロセスを知らない。バックグラウンドプロセスは残り続け、後続のテストに干渉する可能性がある。

### Solution

**E2E test subagent を dispatch する前に、prompt に cleanup を含める:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- 古いプロセスが誤った config でリクエストを処理する
- port conflict が silent failure を引き起こす
- プロセスの蓄積が system を遅くする
- 混乱を招く test result（誤った server に当たる）
```

**トレードオフ分析:**
- prompt に boilerplate が増える
- しかし非常に混乱しやすいデバッグを防げる
- E2E test subagent には見合う価値がある

---

### 3. subagent-driven-development: Lean Context Option を追加する

**Step 2: Execute Task with Subagent を変更**

**Before:**
```
Read that task carefully from [plan-file].
```

**After:**
```
## Context Approaches

**Full Plan (default):**
タスクが複雑、または依存関係がある場合に使う:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
タスクが独立しており、pattern-based な場合に使う:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- タスクが既存パターンに従う（類似テストの追加、類似機能の実装）
- タスクが self-contained である（他タスクの context を必要としない）
- パターン参照で十分である（例: "follow TestE2E_FeatureOptionValidation"）

**Use full plan when:**
- タスクが他タスクに依存している
- 全体アーキテクチャの理解が必要
- context を要する複雑な logic がある
```

**例:**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**これが有効な理由:**
token 使用量を減らし、集中度を高め、適切な場面ではより速い完了につながる。

---

### 4. subagent-driven-development: Self-Reflection Step を追加する

**Step 2: Execute Task with Subagent を変更**

**prompt template に追加:**

```
完了したら、報告する BEFORE に:

一歩引いて、新鮮な目で自分の作業を見直してください。

自分に問いかけること:
- これは本当に、指定されたとおりにタスクを解決しているか？
- 見落とした edge case はないか？
- パターンに正しく従ったか？
- tests が失敗しているなら、ROOT CAUSE は何か（implementation bug か test bug か）？
- この implementation には何をもっと良くできるか？

この振り返りで問題を見つけたら、今ここで修正してください。

そのうえで報告する内容:
- 実装したこと
- 自己振り返りの所見（あれば）
- Test results
- 変更した files
```

**これが有効な理由:**
handoff 前に、implementer 自身が見つけられるバグを捕まえられる。実例あり: 自己振り返りによって entrypoint bug を特定した。

**トレードオフ:**
タスクごとに約 30 秒増えるが、review 前に問題を捕まえられる。

---

### 5. requesting-code-review: 明示的な File Reading を追加する

**code-reviewer template を変更:**

**冒頭に追加:**

```markdown
## Files to Review

分析を始める BEFORE に、これらの files を読んでください:

1. [diff で変更された specific files の一覧]
2. [変更で参照されるが修正されていない files]

Use Read tool to load each file.

もし file を見つけられない場合:
- diff の正確な path を確認する
- 別の場所も試す
- 次を報告する: "Cannot locate [path] - please verify file exists"

実際の code を読むまで、review を進めてはいけません。
```

**これが有効な理由:**
明示的な指示により、「file not found」の問題を防げる。

---

### 6. testing-anti-patterns: Mock-Interface Drift Anti-Pattern を追加する

**新しい Anti-Pattern 6 を追加:**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock が bug を test に埋め込んでしまう
- TypeScript は、誤った method name を持つ inline mock を検出できない
- code も mock も誤っているため、test が pass してしまう
- 実際の object を使うと runtime で crash する

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
mock を書く BEFORE に:

  1. STOP - まだ code under test を見ない
  2. FIND: dependency の interface/type definition を探す
  3. READ: interface file を読む
  4. LIST: interface で定義されている methods を列挙する
  5. MOCK: その methods だけを、その名前どおりに EXACTLY mock する
  6. DO NOT: 自分の code が何を呼んでいるかを見ない

  test が、code が mock にないものを呼んでいるせいで失敗した場合:
    ✅ GOOD - test が code の bug を見つけた
    code を修正して、正しい interface method を呼ぶようにする
    mock ではなく

  Red flags:
    - "I'll mock what the code calls"
    - implementation から method name をコピーする
    - interface を読まずに mock を書く
    - "The test is failing so I'll add this method to the mock"
```

**Detection:**

runtime error の "X is not a function" が出て tests が pass している場合:
1. X が mock されているか確認する
2. mock methods と interface methods を比較する
3. method name の不一致を探す
```

**これが有効な理由:**
フィードバックで報告された失敗パターンに直接対応しているため。

---

### 7. subagent-driven-development: Test Subagents に Skills Reading を必須化する

**task が testing を含む場合、prompt template に追加:**

```markdown
tests を書く BEFORE に:

1. testing-anti-patterns skill を読む:
   Use Skill tool: superpowers:testing-anti-patterns

2. この skill の gate functions を次の場合に適用する:
   - mocks を書くとき
   - production classes に methods を追加するとき
   - dependencies を mocking するとき

これは任意ではありません。anti-patterns に違反した tests は review で reject されます。
```

**これが有効な理由:**
スキルが単に存在するだけでなく、実際に使われるようにするため。

**トレードオフ:**
各 task に時間が追加されるが、バグのクラス全体を防げる。

---

### 8. subagent-driven-development: Implementer が自己特定した問題を修正できるようにする

**Step 2 を変更:**

**Current:**
```
Subagent reports back with summary of work.
```

**Proposed:**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**これが有効な理由:**
Implementer がすでに fix を把握している場合の latency を減らせる。実例あり: entrypoint bug で 1 往復減らせたはず。

**トレードオフ:**
prompt は少し複雑になるが、end-to-end は速くなる。

---

## 実装計画

### Phase 1: High-Impact, Low-Risk（最初に実施）

1. **verification-before-completion: Configuration change verification**
   - 明確な追加であり、既存内容を変更しない
   - 影響の大きい問題（テストへの誤った安心感）に対応する
   - File: `skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns: Mock-interface drift**
   - 新しい anti-pattern を追加するが、既存内容は変更しない
   - 影響の大きい問題（runtime crash）に対応する
   - File: `skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review: Explicit file reading**
   - template への単純な追加
   - 具体的な問題（reviewer が files を見つけられない）を修正する
   - File: `skills/requesting-code-review/SKILL.md`

### Phase 2: Moderate Changes（慎重にテスト）

4. **subagent-driven-development: Process hygiene**
   - 新しい section を追加するが、workflow は変えない
   - 中〜高影響の問題（test reliability）に対応する
   - File: `skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development: Self-reflection**
   - prompt template を変更する（高リスク）
   - ただし bug を捕まえる実例がある
   - File: `skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development: Skills reading requirement**
   - prompt overhead を増やす
   - ただしスキルが実際に使われることを保証する
   - File: `skills/subagent-driven-development/SKILL.md`

### Phase 3: Optimization（先に検証する）

7. **subagent-driven-development: Lean context option**
   - 複雑さが増す（2 つのアプローチ）
   - 混乱を招かないことの検証が必要
   - File: `skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development: Allow implementer to fix**
   - workflow を変更する（高リスク）
   - bug fix ではなく最適化
   - File: `skills/subagent-driven-development/SKILL.md`

---

## 未解決の問い

1. **Lean context approach:**
   - pattern-based な task では、これを default にすべきか？
   - どのように使い分けを判断するか？
   - lean にしすぎて重要な context を見落とすリスクは？

2. **Self-reflection:**
   - 単純な task でも大きく遅くなるか？
   - 複雑な task にだけ適用すべきか？
   - 形式化してしまう "reflection fatigue" をどう防ぐか？

3. **Process hygiene:**
   - これは subagent-driven-development に入れるべきか、それとも別 skill か？
   - E2E tests 以外の workflow にも適用されるか？
   - プロセスが残るべきケース（dev servers）はどう扱うか？

4. **Skills reading enforcement:**
   - すべての subagent に関連 skill の読解を必須化すべきか？
   - prompt が長くなりすぎないようにするには？
   - 文書化しすぎて集中を失うリスクは？

---

## 成功指標

これらの改善が機能していると、どう判断するか？

1. **Configuration verification:**
   - 「test は pass したが誤った config が使われていた」事例がゼロ
   - Jesse が「それは実際には思っているものをテストしていない」と言わない

2. **Process hygiene:**
   - 「test が誤った server に当たった」事例がゼロ
   - E2E test 実行中に port conflict error が発生しない

3. **Mock-interface drift:**
   - 「tests は pass するが runtime で method 不足により crash する」事例がゼロ
   - mocks と interfaces の間で method name の不一致がない

4. **Self-reflection:**
   - 測定可能な点: implementer の報告に self-reflection findings が含まれるか？
   - 定性的な点: code review に流れ込む bug が減るか？

5. **Skills reading:**
   - subagent の報告が skill の gate functions を参照している
   - code review で anti-pattern 違反が減る

---

## リスクと緩和策

### リスク: Prompt Bloat
**問題:** これらすべての要件を追加すると prompt が過剰になる
**緩和策:**
- 段階的に実装する（一度に全部追加しない）
- いくつかの追加は条件付きにする（E2E hygiene は E2E tests のみ）
- task type ごとの template を検討する

### リスク: Analysis Paralysis
**問題:** 振り返り / 検証が多すぎて実行が遅くなる
**緩和策:**
- gate functions は短時間で済むようにする（数分ではなく数秒）
- lean context は最初は opt-in にする
- task completion time を監視する

### リスク: False Sense of Security
**問題:** checklist に従っても正しさは保証されない
**緩和策:**
- gate functions は最低限であり上限ではないと強調する
- skills に "use judgment" の文言を残す
- skills は一般的な失敗を捕まえるものであり、すべてではないと明記する

### リスク: Skill Divergence
**問題:** 異なる skills が矛盾する助言をする
**緩和策:**
- すべての skills を横断して一貫性を確認する
- skills の相互作用を文書化する（Integration sections）
- 本番投入前に実シナリオでテストする

---

## 推奨事項

**Phase 1 をただちに進める:**
- verification-before-completion: Configuration change verification
- testing-anti-patterns: Mock-interface drift
- requesting-code-review: Explicit file reading

**最終確定前に Jesse とともに Phase 2 をテストする:**
- self-reflection の影響についてフィードバックを得る
- process hygiene approach を検証する
- skills reading requirement が overhead に見合うか確認する

**Phase 3 は検証完了まで保留する:**
- lean context は実運用でのテストが必要
- implementer-fix workflow change は慎重な評価が必要

これらの変更は、skills を悪化させるリスクを最小化しつつ、ユーザーによって記録された実際の問題に対処している。
