# Svelte Todo List - 設計

## 概要

Svelte で構築するシンプルな todo list アプリケーションです。todo の作成、完了、削除をサポートし、`localStorage` に永続化します。

## 機能

- 新しい todo を追加
- todo を完了/未完了に切り替え
- todo を削除
- フィルター: All / Active / Completed
- 完了済み todo をすべて削除
- `localStorage` に永続化
- 残り件数を表示

## ユーザーインターフェース

```
┌─────────────────────────────────────────┐
│  Svelte Todos                           │
├─────────────────────────────────────────┤
│  [________________________] [Add]       │
├─────────────────────────────────────────┤
│  [ ] Buy groceries                  [x] │
│  [✓] Walk the dog                   [x] │
│  [ ] Write code                     [x] │
├─────────────────────────────────────────┤
│  2 items left                           │
│  [All] [Active] [Completed]  [Clear ✓]  │
└─────────────────────────────────────────┘
```

## コンポーネント

```
src/
  App.svelte           # メイン app、state 管理
  lib/
    TodoInput.svelte   # テキスト入力 + Add ボタン
    TodoList.svelte    # リストのコンテナ
    TodoItem.svelte    # checkbox、テキスト、削除を持つ単一 todo
    FilterBar.svelte   # filter ボタン + clear completed
    store.ts           # todo 用の Svelte store
    storage.ts         # localStorage 永続化
```

## データモデル

```typescript
interface Todo {
  id: string;        // UUID
  text: string;      // Todo テキスト
  completed: boolean;
}

type Filter = 'all' | 'active' | 'completed';
```

## 受け入れ条件

1. 入力して Enter を押すか Add をクリックすると todo を追加できる
2. checkbox をクリックすると todo の完了状態を切り替えられる
3. X ボタンをクリックすると todo を削除できる
4. filter ボタンで正しい todo の部分集合が表示される
5. `X items left` に未完了 todo の件数が表示される
6. `Clear completed` ですべての完了済み todo が削除される
7. page を更新しても todo が保持される（`localStorage`）
8. 空の状態では分かりやすいメッセージを表示する
9. すべてのテストが通る
