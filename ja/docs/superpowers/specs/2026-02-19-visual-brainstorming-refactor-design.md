# Visual Brainstorming リファクタ: ブラウザ表示とターミナルコマンド

**Date:** 2026-02-19
**Status:** 承認済み
**Scope:** `lib/brainstorm-server/`, `skills/brainstorming/visual-companion.md`, `tests/brainstorm-server/`

## 問題

視覚的なブレインストーミング中、Claude は `wait-for-feedback.sh` をバックグラウンドタスクとして実行し、`TaskOutput(block=true, timeout=600s)` でブロックします。これにより TUI 全体が占有され、visual brainstorming の実行中はユーザーが Claude に入力できなくなります。ブラウザが唯一の入力チャネルになってしまいます。

Claude Code の実行モデルはターンベースです。1 回のターンの中で Claude が 2 つのチャネルを同時に待ち受ける方法はありません。ブロッキングな `TaskOutput` パターンは誤ったプリミティブでした。これは、プラットフォームがサポートしていないイベント駆動の振る舞いを擬似的に再現していたのです。

## 設計

### コアモデル

**ブラウザ = 対話型ディスプレイ。** モックアップを表示し、ユーザーがクリックして選択肢を選べるようにします。選択内容はサーバー側で記録されます。

**ターミナル = 会話チャネル。** 常に非ブロックで、常に利用可能です。ユーザーはここで Claude と会話します。

### ループ

1. Claude がセッションディレクトリに HTML ファイルを書き込む
2. サーバーが chokidar でそれを検出し、WebSocket の reload をブラウザへ送る（変更なし）
3. Claude はターンを終了し、ブラウザを確認してターミナルで返答するようユーザーに伝える
4. ユーザーはブラウザを見て、必要ならクリックして選択肢を選び、その後ターミナルでフィードバックを入力する
5. 次のターンで Claude はブラウザ操作ストリーム（クリック、選択）を得るために `$SCREEN_DIR/.events` を読み、ターミナルのテキストと統合する
6. 反復または前進する

バックグラウンドタスクはありません。`TaskOutput` のブロッキングもありません。ポーリング用スクリプトもありません。

### 主要な削除: `wait-for-feedback.sh`

完全に削除します。その役割は「サーバーがイベントを stdout に記録する」と「Claude がそれらのイベントを受け取る必要がある」を橋渡しすることでした。これを `.events` ファイルが置き換えます。サーバーはユーザー操作イベントを直接書き込み、Claude はプラットフォームが提供する任意のファイル読み取り機構でそれを読みます。

### 主要な追加: `.events` ファイル（画面ごとのイベントストリーム）

サーバーはすべてのユーザー操作イベントを `$SCREEN_DIR/.events` に 1 行 1 JSON オブジェクトで書き込みます。これにより Claude は、現在の画面における完全な操作ストリームを取得できます。最終的な選択だけでなく、ユーザーがどのように探索したか（A をクリックし、次に B、最終的に C に落ち着いた）も分かります。

ユーザーが選択肢を探索した後の内容例:

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 1 画面内では追記専用です。各ユーザーイベントは新しい行として追加されます。
- 新しい HTML ファイル（新しい画面の push）を chokidar が検出したら、このファイルはクリア（削除）され、古いイベントが持ち越されるのを防ぎます。
- Claude が読む時点でファイルが存在しなければ、ブラウザ操作は発生していません。その場合 Claude はターミナルのテキストだけを使います。
- このファイルにはユーザーイベント（`click` など）のみを含め、サーバーのライフサイクルイベント（`server-started`, `screen-added`）は含めません。これにより小さく、目的に集中したものになります。
- Claude はユーザーの探索パターンを理解するために完全なストリームを読んでもよく、最終選択だけ必要なら最後の `choice` イベントだけを見ても構いません。

## ファイルごとの変更

### `index.js`（server）

**A. ユーザーイベントを `.events` ファイルに書き込む。**

WebSocket の `message` ハンドラーで、イベントを stdout に記録した後、`fs.appendFileSync` を使ってイベントを JSON Lines として `$SCREEN_DIR/.events` に追記します。書き込むのはユーザー操作イベント（`source: 'user-event'` を持つもの）のみで、サーバーのライフサイクルイベントは書き込みません。

**B. 新しい画面で `.events` をクリアする。**

chokidar の `add` ハンドラー（新しい `.html` ファイルを検出）で、存在するなら `$SCREEN_DIR/.events` を削除します。これは明確な「新しい画面」のシグナルであり、毎回の reload で発火する GET `/` でクリアするより適切です。

**C. `wrapInFrame` の content injection を置き換える。**

現在の regex は `<div class="feedback-footer">` を基準にしていますが、これは削除されます。代わりにコメントプレースホルダーを使います。`#claude-content` 内の既存デフォルトコンテンツ（`<h2>Visual Brainstorming</h2>` とサブタイトル段落）を削除し、単一の `<!-- CONTENT -->` マーカーに置き換えます。content injection は `frameTemplate.replace('<!-- CONTENT -->', content)` になります。より単純で、テンプレートの整形が変わっても壊れません。

### `frame-template.html`（UI frame）

**削除:**
- `feedback-footer` div（textarea、Send button、label、`.feedback-row`）
- 関連 CSS（`.feedback-footer`, `.feedback-footer label`, `.feedback-row`, その中の textarea と button のスタイル）

**追加:**
- `#claude-content` 内に `<!-- CONTENT -->` プレースホルダーを追加し、デフォルトテキストを置き換える
- フッターがあった場所に選択状態インジケータバーを追加し、2 つの状態を持たせる:
  - デフォルト: "上のオプションをクリックしてから、ターミナルに戻ってください"
  - 選択後: "Option B を選択しました — 続行するにはターミナルに戻ってください"
- インジケータバー用の CSS（控えめで、既存ヘッダーに近い視覚的な重み）

**変更しないもの:**
- "Brainstorm Companion" タイトルと接続状態を持つヘッダーバー
- `.main` ラッパーと `#claude-content` コンテナ
- すべてのコンポーネント CSS（`.options`, `.cards`, `.mockup`, `.split`, `.pros-cons`, placeholders, mock elements）
- ダーク/ライトテーマ変数と media query

### `helper.js`（client-side script）

**削除:**
- `sendToClaude()` 関数と "Sent to Claude" の全画面 takeover
- `window.send()` 関数（削除される Send button に結び付いていた）
- フォーム送信ハンドラー — フィードバック textarea がなくなるため用途がなく、ログノイズを増やすだけ
- input change ハンドラー — 同上
- `pageshow` event listener（textarea の内容保持を修正するために追加されていたが、textarea はもうない）

**維持:**
- WebSocket 接続、再接続ロジック、イベントキュー
- reload ハンドラー（サーバー push 時の `window.location.reload()`）
- 選択状態をハイライトする `window.toggleSelect()`
- `window.selectedChoice` の追跡
- `window.brainstorm.send()` と `window.brainstorm.choice()` — これらは削除される `window.send()` とは別物です。`sendEvent` を呼んで WebSocket 経由でサーバーへ記録します。カスタム full-document page に有用です。

**狭める:**
- クリックハンドラーは、すべての button/link ではなく `[data-choice]` クリックだけを捕捉するようにします。広範な捕捉が必要だったのは、ブラウザがフィードバックチャネルだった頃の話で、今は選択追跡だけが目的です。

**追加:**
- `data-choice` クリック時、どの option が選ばれたかを表示するように選択状態インジケータバーのテキストを更新する

**`window.brainstorm` API から削除:**
- `brainstorm.sendToClaude` — もう存在しません

### `visual-companion.md`（skill instructions）

**"The Loop" セクションを書き換える** ことで、上記の非ブロッキングフローを反映します。以下への言及はすべて削除します:
- `wait-for-feedback.sh`
- `TaskOutput` のブロッキング
- timeout/retry ロジック（600s timeout、30 分上限）
- `send-to-claude` JSON を説明する "User Feedback Format" セクション

**代わりに入れる内容:**
- 新しいループ（HTML を書く → ターンを終える → ユーザーがターミナルで返答 → `.events` を読む → 反復）
- `.events` ファイル形式のドキュメント
- ターミナルメッセージが主たるフィードバックであり、`.events` は補助的な文脈として完全なブラウザ操作ストリームを提供する、というガイダンス

**維持:**
- サーバー起動/停止手順
- content fragment と full document に関するガイダンス
- CSS class リファレンスと利用可能なコンポーネント
- デザインのコツ（質問に応じて忠実度を調整する、1 画面あたり 2〜4 オプションなど）

### `wait-for-feedback.sh`

**完全に削除。**

### `tests/brainstorm-server/server.test.js`

更新が必要なテスト:
- fragment response に `feedback-footer` が存在することを確認しているテスト — 選択状態インジケータバー、または `<!-- CONTENT -->` 置換を確認するよう更新
- `helper.js` に `send` が含まれることを確認しているテスト — API を狭めた内容に合わせて更新
- `sendToClaude` の CSS variable 使用を確認しているテスト — 削除（関数自体が存在しないため）

## プラットフォーム互換性

サーバーコード（`index.js`, `helper.js`, `frame-template.html`）は完全にプラットフォーム非依存です。純粋な Node.js とブラウザ JavaScript のみで、Claude Code 固有の参照はありません。バックグラウンドのターミナル操作を通じて Codex 上でも動作することはすでに確認済みです。

skill instructions（`visual-companion.md`）がプラットフォーム適応層です。各プラットフォーム上の Claude は、それぞれ独自のツールでサーバー起動や `.events` の読み取りなどを行います。非ブロッキングモデルは、プラットフォーム固有のブロッキングプリミティブに依存しないため、自然にクロスプラットフォームで機能します。

## これによって実現されること

- **visual brainstorming 中でも TUI が常に応答可能**
- **混在入力** — ブラウザでクリック + ターミナルで入力、を自然に統合
- **Graceful degradation** — ブラウザが落ちている、またはユーザーが開かない場合でも、ターミナルは引き続き機能する
- **より単純なアーキテクチャ** — バックグラウンドタスクなし、ポーリングスクリプトなし、timeout 管理なし
- **クロスプラットフォーム** — 同じサーバーコードが Claude Code、Codex、将来のあらゆるプラットフォームで動作する

## これによって失われるもの

- **ブラウザだけで完結するフィードバックワークフロー** — 続行するにはユーザーがターミナルへ戻る必要があります。選択状態インジケータバーがそれを案内しますが、以前の「クリック → Send → 待機」フローに比べると 1 ステップ増えます。
- **ブラウザからのインラインテキストフィードバック** — textarea は廃止されます。テキストフィードバックはすべてターミナル経由です。これは意図的なもので、フレーム内の小さな textarea よりターミナルの方がテキスト入力チャネルとして優れています。
- **ブラウザの Send に対する即時応答** — 旧システムではユーザーが Send をクリックした瞬間に Claude が応答していました。新システムでは、ユーザーがターミナルへ切り替える間に小さなギャップがあります。実際には数秒で、しかもユーザーはターミナルメッセージに追加の文脈を書けます。
