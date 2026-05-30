# 多層防御による Validation

## 概要

無効なデータが原因のバグを直すとき、1 箇所に validation を追加すれば十分に思える。しかし、その単一チェックは別の code path や refactoring、mocks によって回避されうる。

**中核原則:** データが通過する**すべての**層で validation すること。バグを構造的に不可能にする。

## なぜ複数層なのか

単一の validation: 「バグを直した」
複数層: 「バグを不可能にした」

異なる層は異なるケースを捕まえる:
- Entry validation は大半のバグを捕まえる
- Business logic はエッジケースを捕まえる
- Environment guards は文脈依存の危険を防ぐ
- Debug logging は他の層が失敗したときに役立つ

## 4 つの層

### Layer 1: Entry Point Validation
**目的:** API 境界で明らかに無効な入力を拒否する

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... proceed
}
```

### Layer 2: Business Logic Validation
**目的:** この操作に対してデータが意味をなすことを保証する

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### Layer 3: Environment Guards
**目的:** 特定の文脈で危険な操作を防ぐ

```typescript
async function gitInit(directory: string) {
  // In tests, refuse git init outside temp directories
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... proceed
}
```

### Layer 4: Debug Instrumentation
**目的:** フォレンジックのための文脈を記録する

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... proceed
}
```

## パターンの適用方法

バグを見つけたら:

1. **データフローを追跡する** - bad value はどこで生まれ、どこで使われるか？
2. **すべての checkpoint を洗い出す** - データが通るすべての地点を列挙する
3. **各層に validation を追加する** - entry、business、environment、debug
4. **各層をテストする** - layer 1 を回避してみて、layer 2 が捕まえることを確認する

## セッションからの例

バグ: 空の `projectDir` によりソースコード内で `git init` が実行された

**データフロー:**
1. テストセットアップ → 空文字列
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` が `process.cwd()` で実行される

**追加した 4 層:**
- Layer 1: `Project.create()` が empty / exists / writable を検証
- Layer 2: `WorkspaceManager` が `projectDir` が空でないことを検証
- Layer 3: `WorktreeManager` が tests 中に tmpdir 外での `git init` を拒否
- Layer 4: `git init` 前に stack trace logging

**結果:** 1847 件のテストがすべて通過し、バグは再現不可能になった

## 重要な洞察

4 層すべてが必要だった。テスト中、各層は他の層が見逃したバグを捕まえた:
- 別の code paths が entry validation を回避した
- mocks が business logic の checks を回避した
- 別プラットフォームでのエッジケースには environment guards が必要だった
- Debug logging が構造的な誤用を特定した

**validation を 1 箇所で止めてはならない。** すべての層に checks を追加すること。
