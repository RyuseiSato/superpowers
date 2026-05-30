# Zero-Dependency Brainstorm Server

brainstorm companion server の vendored node_modules（express、ws、chokidar — 714 個の追跡ファイル）を、Node.js built-ins だけを使う単一の zero-dependency `server.js` に置き換えます。

## 動機

node_modules を git リポジトリに vendor することは、サプライチェーン上のリスクを生みます。凍結された依存関係にはセキュリティパッチが適用されず、714 ファイル分のサードパーティコードが監査なしでコミットされ、vendored code への変更は通常のコミットと見分けがつきません。実際のリスクは低い（localhost 専用の dev server）ものの、これをなくすのは簡単です。

## アーキテクチャ

`http`, `crypto`, `fs`, `path` を使う単一の `server.js` ファイル（約 250〜300 行）。このファイルは 2 つの役割を持ちます。

- **直接実行時**（`node server.js`）: HTTP/WebSocket server を起動する
- **require された時**（`require('./server.js')`）: 単体テスト向けに WebSocket protocol 関数を export する

### WebSocket Protocol

RFC 6455 を text frame のみについて実装します。

**Handshake:** クライアントの `Sec-WebSocket-Key` から、SHA-1 と RFC 6455 の magic GUID を使って `Sec-WebSocket-Accept` を計算します。101 Switching Protocols を返します。

**Frame decoding（client から server）:** 3 種類の masked length encoding を扱います。
- Small: payload が 126 bytes 未満
- Medium: 126-65535 bytes（16-bit extended）
- Large: 65535 bytes 超（64-bit extended）

4-byte の mask key を使って payload を XOR-unmask します。`{ opcode, payload, bytesConsumed }` を返し、buffer が不完全な場合は `null` を返します。unmasked frames は拒否します。

**Frame encoding（server から client）:** 同じ 3 種類の length encoding を持つ unmasked frames。

**扱う Opcodes:** TEXT (`0x01`), CLOSE (`0x08`), PING (`0x09`), PONG (`0x0A`)。認識できない opcode には status 1003（Unsupported Data）の close frame を返します。

**意図的に省くもの:** Binary frames、fragmented messages、extensions（permessage-deflate）、subprotocols。localhost clients 間でやり取りする小さな JSON text messages には不要です。extensions と subprotocols は handshake でネゴシエートされるため、こちらが広告しなければ有効になりません。

**Buffer accumulation:** 各接続は buffer を保持します。`data` 時に追記し、`decodeFrame` が `null` を返すか buffer が空になるまでループします。

### HTTP Server

ルートは 3 つです。

1. **`GET /`** — screen directory から mtime が最新の `.html` を返します。full document と fragment を判定し、fragment は frame template で包んで helper.js を注入します。`text/html` を返します。`.html` ファイルが存在しない場合は、helper.js を注入したハードコードの waiting page（"Waiting for Claude to push a screen..."）を返します。
2. **`GET /files/*`** — screen directory から静的ファイルを返します。MIME type はハードコードされた拡張子マップ（html, css, js, png, jpg, gif, svg, json）で判定します。見つからなければ 404 を返します。
3. **その他すべて** — 404。

WebSocket upgrade は HTTP server の request handler とは別に、`'upgrade'` event で処理します。

### 設定

環境変数（すべて任意）:

- `BRAINSTORM_PORT` — bind する port（デフォルト: 49152-65535 のランダムな高位ポート）
- `BRAINSTORM_HOST` — bind する interface（デフォルト: `127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 起動 JSON 内の URL 用 hostname（デフォルト: host が `127.0.0.1` の場合は `localhost`、それ以外は host と同じ）
- `BRAINSTORM_DIR` — screen directory path（デフォルト: `/tmp/brainstorm`）

### 起動シーケンス

1. `SCREEN_DIR` が存在しなければ作成する（`mkdirSync` recursive）
2. frame template と helper.js を `__dirname` から読み込む
3. 設定された host/port で HTTP server を起動する
4. `SCREEN_DIR` に対して `fs.watch` を開始する
5. listen 成功時、`server-started` JSON を stdout に記録する: `{ type, port, host, url_host, url, screen_dir }`
6. 同じ JSON を `SCREEN_DIR/.server-info` に書き込み、stdout が隠れている場合（background execution）でも agent が接続情報を見つけられるようにする

### アプリケーションレベルの WebSocket Messages

client から TEXT frame が届いたとき:

1. JSON として parse する。parse に失敗したら stderr に記録して継続する。
2. `{ source: 'user-event', ...event }` として stdout に記録する。
3. event に `choice` プロパティが含まれていれば、その JSON を `SCREEN_DIR/.events` に追記する（1 event につき 1 行）。

### ファイル監視

`fs.watch(SCREEN_DIR)` が chokidar を置き換えます。HTML ファイルイベント時の挙動:

- 新規ファイル（存在するファイルに対する `rename` event）: `.events` ファイルがあれば削除し（`unlinkSync`）、`screen-added` を JSON として stdout に記録する
- ファイル変更（`change` event）: `screen-updated` を JSON として stdout に記録する（`.events` はクリアしない）
- どちらのイベントでも: 接続中の全 WebSocket client に `{ type: 'reload' }` を送る

重複イベントを防ぐため、filename ごとに約 100ms の debounce を入れます（macOS と Linux では一般的）。

### エラーハンドリング

- WebSocket client からの不正な JSON: stderr に記録し、継続
- 未処理の opcodes: status 1003 で close
- client 切断: broadcast set から削除
- `fs.watch` エラー: stderr に記録し、継続
- graceful shutdown logic はなし — process lifecycle は shell scripts が SIGTERM で扱う

## 何が変わるか

| Before | After |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 個の `node_modules` ファイル | `server.js`（単一ファイル） |
| express, ws, chokidar dependencies | なし |
| 静的ファイル配信なし | `/files/*` が screen directory から配信 |

## 変わらないもの

- `helper.js` — 変更なし
- `frame-template.html` — 変更なし
- `start-server.sh` — 1 行だけ更新: `index.js` を `server.js` に変更
- `stop-server.sh` — 変更なし
- `visual-companion.md` — 変更なし
- 既存の server behavior と external contract はすべて維持

## プラットフォーム互換性

- `server.js` はクロスプラットフォームな Node built-ins だけを使う
- `fs.watch` は macOS、Linux、Windows の単一フラットディレクトリでは信頼できる
- shell scripts には bash が必要（Windows では Git Bash。これは Claude Code の必須要件）

## テスト

**Unit tests**（`ws-protocol.test.js`）: `server.js` exports を require して、WebSocket frame の encode/decode、handshake 計算、protocol edge cases を直接テストします。

**Integration tests**（`server.test.js`）: 完全な server behavior — HTTP serving、WebSocket communication、file watching、brainstorming workflow — をテストします。テスト専用の client dependency として `ws` npm package を使用します（end user には配布しません）。
