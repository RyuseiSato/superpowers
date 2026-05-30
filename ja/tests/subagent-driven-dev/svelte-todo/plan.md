# Svelte Todo List - 実装計画

この計画は `superpowers:subagent-driven-development` skill を使って実行してください。

## Context

Svelte で todo list app を構築します。完全な仕様は `design.md` を参照してください。

## Tasks

### Task 1: Project Setup

Vite を使って Svelte project を作成します。

**実施内容:**
- `npm create vite@latest . -- --template svelte-ts` を実行
- `npm install` で依存関係をインストール
- dev server が動作することを確認
- `App.svelte` からデフォルトの Vite template 内容を削除

**確認:**
- `npm run dev` で server が起動する
- App に最小限の `Svelte Todos` 見出しが表示される
- `npm run build` が成功する

---

### Task 2: Todo Store

Todo state 管理用の Svelte store を作成します。

**実施内容:**
- `src/lib/store.ts` を作成
- `id`, `text`, `completed` を持つ `Todo` interface を定義
- 初期値が空配列の writable store を作成
- `addTodo(text)`, `toggleTodo(id)`, `deleteTodo(id)`, `clearCompleted()` を export
- 各関数のテストを持つ `src/lib/store.test.ts` を作成

**確認:**
- テストが通る: `npm run test`（必要なら vitest をインストール）

---

### Task 3: localStorage Persistence

Todo の永続化レイヤーを追加します。

**実施内容:**
- `src/lib/storage.ts` を作成
- `loadTodos(): Todo[]` と `saveTodos(todos: Todo[])` を実装
- JSON parse error を適切に処理する（空配列を返す）
- store と統合: 初期化時に読み込み、変更時に保存
- load/save/error handling のテストを追加

**確認:**
- テストが通る
- 手動テスト: todo を追加して page を更新しても保持される

---

### Task 4: TodoInput Component

Todo を追加する入力 component を作成します。

**実施内容:**
- `src/lib/TodoInput.svelte` を作成
- テキスト入力をローカル state に bind
- Add ボタンで `addTodo()` を呼び、入力をクリア
- Enter キーでも submit
- 入力が空のとき Add ボタンを無効化
- component テストを追加

**確認:**
- テストが通る
- component に input と button が表示される

---

### Task 5: TodoItem Component

単一の todo item component を作成します。

**実施内容:**
- `src/lib/TodoItem.svelte` を作成
- Props: `todo: Todo`
- Checkbox で完了状態を切り替える（`toggleTodo` を呼ぶ）
- 完了時は取り消し線付きのテキスト
- Delete ボタン（X）で `deleteTodo` を呼ぶ
- component テストを追加

**確認:**
- テストが通る
- component に checkbox、text、delete button が表示される

---

### Task 6: TodoList Component

リスト用のコンテナ component を作成します。

**実施内容:**
- `src/lib/TodoList.svelte` を作成
- Props: `todos: Todo[]`
- 各 todo に対して `TodoItem` を描画
- 空の場合は `No todos yet` を表示
- component テストを追加

**確認:**
- テストが通る
- component が `TodoItem` のリストを描画する

---

### Task 7: FilterBar Component

Filter と status bar の component を作成します。

**実施内容:**
- `src/lib/FilterBar.svelte` を作成
- Props: `todos: Todo[]`, `filter: Filter`, `onFilterChange: (f: Filter) => void`
- 件数を表示: `X items left`（未完了件数）
- 3 つの filter button: All、Active、Completed
- 現在の filter は視覚的にハイライト
- `Clear completed` button（完了済み todo がない場合は非表示）
- component テストを追加

**確認:**
- テストが通る
- component に件数、filters、clear button が表示される

---

### Task 8: App Integration

すべての component を `App.svelte` で接続します。

**実施内容:**
- すべての component と store を import
- filter state を追加（デフォルト: `all`）
- filter state に応じて filtered todos を計算
- 見出し、`TodoInput`、`TodoList`、`FilterBar` を描画
- 各 component に適切な props を渡す

**確認:**
- App にすべての component が表示される
- todo の追加が動作する
- 完了切り替えが動作する
- 削除が動作する

---

### Task 9: Filter Functionality

filtering が end-to-end で動作することを確認します。

**実施内容:**
- filter button で表示中の todo が変わることを確認
- `all` はすべての todo を表示
- `active` は未完了 todo のみ表示
- `completed` は完了済み todo のみ表示
- `Clear completed` で完了済み todo を削除し、必要なら filter を reset
- integration テストを追加

**確認:**
- filter テストが通る
- すべての filter 状態を手動確認

---

### Task 10: Styling and Polish

使いやすさのための CSS styling を追加します。

**実施内容:**
- design mockup に合わせて app を styling
- 完了済み todo は取り消し線と控えめな色にする
- 現在の filter button をハイライト
- input に focus styles を追加
- Delete button は hover 時に表示（または mobile では常時表示）
- responsive layout

**確認:**
- App が視覚的に使いやすい
- style が機能を壊していない

---

### Task 11: End-to-End Tests

完全なユーザーフロー用の Playwright テストを追加します。

**実施内容:**
- Playwright をインストール: `npm init playwright@latest`
- `tests/todo.spec.ts` を作成
- 次のフローをテスト:
  - todo を追加
  - todo を完了
  - todo を削除
  - todo を filter
  - 完了済みをクリア
  - 永続化（追加、reload、確認）

**確認:**
- `npx playwright test` が通る

---

### Task 12: README

project を文書化します。

**実施内容:**
- 以下を含む `README.md` を作成:
  - project の説明
  - Setup: `npm install`
  - Development: `npm run dev`
  - Testing: `npm test` と `npx playwright test`
  - Build: `npm run build`

**確認:**
- README が project を正確に説明している
- 手順どおりに動作する
