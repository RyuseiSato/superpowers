# テストのアンチパターン

**このリファレンスを読み込むタイミング:** テストを書いたり変更したりするとき、mocks を追加するとき、または本番コードにテスト専用メソッドを追加したくなったとき。

## 概要

テストは mock の振る舞いではなく、実際の振る舞いを検証しなければならない。mocks は分離のための手段であって、テスト対象そのものではない。

**中核原則:** mock が何をするかではなく、コードが何をするかをテストする。

**厳密な TDD に従えば、これらのアンチパターンは防げる。**

## 鉄則

```
1. NEVER test mock behavior
2. NEVER add test-only methods to production classes
3. NEVER mock without understanding dependencies
```

## アンチパターン 1: mock の振る舞いをテストする

**違反例:**
```typescript
// ❌ BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**なぜ間違っているのか:**
- 検証しているのはコンポーネントではなく mock が動くこと
- mock があれば通り、なければ失敗する
- 実際の振る舞いについては何も分からない

**your human partner の修正:** 「私たちは mock の振る舞いをテストしているのではないか？」

**修正方法:**
```typescript
// ✅ GOOD: Test real component or don't mock it
test('renders sidebar', () => {
  render(<Page />);  // Don't mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### Gate Function

```
BEFORE asserting on any mock element:
  Ask: "Am I testing real component behavior or just mock existence?"

  IF testing mock existence:
    STOP - Delete the assertion or unmock the component

  Test real behavior instead
```

## アンチパターン 2: 本番コード内のテスト専用メソッド

**違反例:**
```typescript
// ❌ BAD: destroy() only used in tests
class Session {
  async destroy() {  // Looks like production API!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// In tests
afterEach(() => session.destroy());
```

**なぜ間違っているのか:**
- 本番クラスがテスト専用コードで汚染される
- 誤って本番で呼ばれると危険
- YAGNI と関心の分離に反する
- オブジェクトのライフサイクルとエンティティのライフサイクルを混同させる

**修正方法:**
```typescript
// ✅ GOOD: Test utilities handle test cleanup
// Session has no destroy() - it's stateless in production

// In test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// In tests
afterEach(() => cleanupSession(session));
```

### Gate Function

```
BEFORE adding any method to production class:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in test utilities instead

  Ask: "Does this class own this resource's lifecycle?"

  IF no:
    STOP - Wrong class for this method
```

## アンチパターン 3: 理解せずに mock する

**違反例:**
```typescript
// ❌ BAD: Mock breaks test logic
test('detects duplicate server', () => {
  // Mock prevents config write that test depends on!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // Should throw - but won't!
});
```

**なぜ間違っているのか:**
- mock したメソッドにはテストが依存する副作用（config の書き込み）があった
- 「安全のため」の過剰な mocking が実際の振る舞いを壊す
- 間違った理由でテストが通る、または不可解に失敗する

**修正方法:**
```typescript
// ✅ GOOD: Mock at correct level
test('detects duplicate server', () => {
  // Mock the slow part, preserve behavior test needs
  vi.mock('MCPServerManager'); // Just mock slow server startup

  await addServer(config);  // Config written
  await addServer(config);  // Duplicate detected ✓
});
```

### Gate Function

```
BEFORE mocking any method:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real method have?"
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (the actual slow/external operation)
    OR use test doubles that preserve necessary behavior
    NOT the high-level method the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Observe what actually needs to happen
    THEN add minimal mocking at the right level

  Red flags:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

## アンチパターン 4: 不完全な mocks

**違反例:**
```typescript
// ❌ BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**なぜ間違っているのか:**
- **部分的な mocks は構造上の前提を隠してしまう** - 自分が知っているフィールドしか mock していない
- **後続のコードは、含めなかったフィールドに依存しているかもしれない** - 失敗が静かに起こる
- **テストは通るのに integration では失敗する** - mock は不完全、実 API は完全
- **偽の安心感** - テストは実際の振る舞いについて何も証明していない

**鉄則:** 目の前のテストで使うフィールドだけではなく、実際に存在する完全なデータ構造を mock すること。

**修正方法:**
```typescript
// ✅ GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

### Gate Function

```
BEFORE creating mock responses:
  Check: "What fields does the real API response contain?"

  Actions:
    1. Examine actual API response from docs/examples
    2. Include ALL fields system might consume downstream
    3. Verify mock matches real response schema completely

  Critical:
    If you're creating a mock, you must understand the ENTIRE structure
    Partial mocks fail silently when code depends on omitted fields

  If uncertain: Include all documented fields
```

## アンチパターン 5: 後回しにされる integration tests

**違反例:**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**なぜ間違っているのか:**
- テストは実装の一部であり、任意の後追い作業ではない
- TDD ならこれを防げた
- テストなしで完了とは言えない

**修正方法:**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## mocks が複雑になりすぎたとき

**警告サイン:**
- mock のセットアップがテスト本体より長い
- テストを通すために何もかも mock している
- mock に実コンポーネントが持つメソッドが欠けている
- mock を変えるとテストが壊れる

**your human partner の質問:** 「ここで本当に mock を使う必要があるか？」

**検討すること:** 複雑な mocks より、実コンポーネントを使った integration tests の方が単純なことが多い

## TDD はこれらのアンチパターンを防ぐ

**TDD が役立つ理由:**
1. **最初にテストを書く** → 何を本当にテストしているのかを考えざるを得ない
2. **失敗するのを見る** → mock ではなく実際の振る舞いをテストしていることを確認できる
3. **最小限の実装** → テスト専用メソッドが入り込まない
4. **実際の依存関係** → mock する前に、テストが本当に何を必要としているか見える

**mock の振る舞いをテストしているなら、TDD に違反している** - 実コード相手に失敗するのを確認する前に mocks を追加したということだ。

## クイックリファレンス

| アンチパターン | 修正 |
|--------------|-----|
| mock 要素に assert する | 実コンポーネントをテストするか、unmock する |
| 本番コードにテスト専用メソッド | テストユーティリティへ移す |
| 理解せずに mock | まず依存関係を理解し、最小限だけ mock する |
| 不完全な mocks | 実 API を完全に再現する |
| 後回しのテスト | TDD - テストを先に |
| 過度に複雑な mocks | integration tests を検討する |

## 危険信号

- assertion が `*-mock` の test ID を確認している
- テストファイルでしか呼ばれないメソッドがある
- mock セットアップがテストの 50% 超を占める
- mock を外すとテストが失敗する
- なぜ mock が必要か説明できない
- 「安全のために」と mock している

## 要点

**mocks は分離のための道具であり、テスト対象そのものではない。**

TDD によって mock の振る舞いをテストしていると分かったなら、道を誤っている。

修正: 実際の振る舞いをテストするか、そもそもなぜ mock しているのかを問い直すこと。
