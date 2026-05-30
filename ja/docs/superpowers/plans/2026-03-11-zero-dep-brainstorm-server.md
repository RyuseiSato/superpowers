# Zero-Dependency Brainstorm Server 実装計画

> **エージェントワーカー向け:** 必須: この計画の実装には superpowers:subagent-driven-development（サブエージェントが利用可能な場合）または superpowers:executing-plans を使用すること。追跡には checkbox (`- [ ]`) syntax を使う。

**目標:** brainstorm server の vendored node_modules を、Node built-ins のみで動く単一の zero-dependency `server.js` に置き換える。

**アーキテクチャ:** WebSocket protocol（RFC 6455 text frames）、HTTP server（`http` module）、file watching（`fs.watch`）を 1 ファイルにまとめる。module として require された場合に unit test できるよう、protocol functions を export する。

**技術スタック:** Node.js built-ins のみ: `http`, `crypto`, `fs`, `path`

**仕様:** `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`

**既存 tests:** `tests/brainstorm-server/ws-protocol.test.js`（unit）、`tests/brainstorm-server/server.test.js`（integration）

---

## File Map

- **Create:** `skills/brainstorming/scripts/server.js` — zero-dep の置き換え実装
- **Modify:** `skills/brainstorming/scripts/start-server.sh:94,100` — `index.js` を `server.js` に変更する
- **Modify:** `.gitignore:6` — `!skills/brainstorming/scripts/node_modules/` 例外を削除する
- **Delete:** `skills/brainstorming/scripts/index.js`
- **Delete:** `skills/brainstorming/scripts/package.json`
- **Delete:** `skills/brainstorming/scripts/package-lock.json`
- **Delete:** `skills/brainstorming/scripts/node_modules/`（714 files）
- **No changes:** `skills/brainstorming/scripts/helper.js`, `skills/brainstorming/scripts/frame-template.html`, `skills/brainstorming/scripts/stop-server.sh`

---

## Chunk 1: WebSocket Protocol Layer

### Task 1: WebSocket protocol exports を実装する

**Files:**
- Create: `skills/brainstorming/scripts/server.js`
- Test: `tests/brainstorm-server/ws-protocol.test.js`（既存）

- [ ] **Step 1: OPCODES 定数と computeAcceptKey を含む server.js を作成する**

```js
const crypto = require('crypto');

const OPCODES = { TEXT: 0x01, CLOSE: 0x08, PING: 0x09, PONG: 0x0A };
const WS_MAGIC = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

function computeAcceptKey(clientKey) {
  return crypto.createHash('sha1').update(clientKey + WS_MAGIC).digest('base64');
}
```

- [ ] **Step 2: encodeFrame を実装する**

server frame は mask されない。長さのエンコーディングは 3 種類:
- payload < 126: 2-byte header（FIN+opcode, length）
- 126-65535: 4-byte header（FIN+opcode, 126, 16-bit length）
- &gt; 65535: 10-byte header（FIN+opcode, 127, 64-bit length）

```js
function encodeFrame(opcode, payload) {
  const fin = 0x80;
  const len = payload.length;
  let header;

  if (len < 126) {
    header = Buffer.alloc(2);
    header[0] = fin | opcode;
    header[1] = len;
  } else if (len < 65536) {
    header = Buffer.alloc(4);
    header[0] = fin | opcode;
    header[1] = 126;
    header.writeUInt16BE(len, 2);
  } else {
    header = Buffer.alloc(10);
    header[0] = fin | opcode;
    header[1] = 127;
    header.writeBigUInt64BE(BigInt(len), 2);
  }

  return Buffer.concat([header, payload]);
}
```

- [ ] **Step 3: decodeFrame を実装する**

client frame は常に mask される。戻り値は `{ opcode, payload, bytesConsumed }`、不完全なら `null`。mask されていない frame では throw する。

```js
function decodeFrame(buffer) {
  if (buffer.length < 2) return null;

  const firstByte = buffer[0];
  const secondByte = buffer[1];
  const opcode = firstByte & 0x0F;
  const masked = (secondByte & 0x80) !== 0;
  let payloadLen = secondByte & 0x7F;
  let offset = 2;

  if (!masked) throw new Error('Client frames must be masked');

  if (payloadLen === 126) {
    if (buffer.length < 4) return null;
    payloadLen = buffer.readUInt16BE(2);
    offset = 4;
  } else if (payloadLen === 127) {
    if (buffer.length < 10) return null;
    payloadLen = Number(buffer.readBigUInt64BE(2));
    offset = 10;
  }

  const maskOffset = offset;
  const dataOffset = offset + 4;
  const totalLen = dataOffset + payloadLen;
  if (buffer.length < totalLen) return null;

  const mask = buffer.slice(maskOffset, dataOffset);
  const data = Buffer.alloc(payloadLen);
  for (let i = 0; i < payloadLen; i++) {
    data[i] = buffer[dataOffset + i] ^ mask[i % 4];
  }

  return { opcode, payload: data, bytesConsumed: totalLen };
}
```

- [ ] **Step 4: ファイル末尾に module exports を追加する**

```js
module.exports = { computeAcceptKey, encodeFrame, decodeFrame, OPCODES };
```

- [ ] **Step 5: unit tests を実行する**

Run: `cd tests/brainstorm-server && node ws-protocol.test.js`
Expected: すべての tests が pass する（handshake、encoding、decoding、boundaries、edge cases）

- [ ] **Step 6: Commit**

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add WebSocket protocol layer for zero-dep brainstorm server"
```

---

## Chunk 2: HTTP Server and Application Logic

### Task 2: HTTP server、file watching、WebSocket connection handling を追加する

**Files:**
- Modify: `skills/brainstorming/scripts/server.js`
- Test: `tests/brainstorm-server/server.test.js`（既存）

- [ ] **Step 1: server.js の先頭（requires の後）に設定と定数を追加する**

```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
const HOST = process.env.BRAINSTORM_HOST || '127.0.0.1';
const URL_HOST = process.env.BRAINSTORM_URL_HOST || (HOST === '127.0.0.1' ? 'localhost' : HOST);
const SCREEN_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm';

const MIME_TYPES = {
  '.html': 'text/html', '.css': 'text/css', '.js': 'application/javascript',
  '.json': 'application/json', '.png': 'image/png', '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg', '.gif': 'image/gif', '.svg': 'image/svg+xml'
};
```

- [ ] **Step 2: WAITING_PAGE、module scope の template 読み込み、helper functions を追加する**

`frameTemplate` と `helperInjection` を module scope に読み込み、`wrapInFrame` と `handleRequest` から参照できるようにする。これらは `__dirname`（scripts directory）からのみファイルを読むため、module として require された場合でも直接実行された場合でも有効である。

```js
const WAITING_PAGE = `<!DOCTYPE html>
<html>
<head><title>Brainstorm Companion</title>
<style>body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
h1 { color: #333; } p { color: #666; }</style>
</head>
<body><h1>Brainstorm Companion</h1>
<p>Waiting for Claude to push a screen...</p></body></html>`;

const frameTemplate = fs.readFileSync(path.join(__dirname, 'frame-template.html'), 'utf-8');
const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
const helperInjection = '<script>\n' + helperScript + '\n</script>';

function isFullDocument(html) {
  const trimmed = html.trimStart().toLowerCase();
  return trimmed.startsWith('<!doctype') || trimmed.startsWith('<html');
}

function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}

function getNewestScreen() {
  const files = fs.readdirSync(SCREEN_DIR)
    .filter(f => f.endsWith('.html'))
    .map(f => {
      const fp = path.join(SCREEN_DIR, f);
      return { path: fp, mtime: fs.statSync(fp).mtime.getTime() };
    })
    .sort((a, b) => b.mtime - a.mtime);
  return files.length > 0 ? files[0].path : null;
}
```

- [ ] **Step 3: HTTP request handler を追加する**

```js
function handleRequest(req, res) {
  if (req.method === 'GET' && req.url === '/') {
    const screenFile = getNewestScreen();
    let html = screenFile
      ? (raw => isFullDocument(raw) ? raw : wrapInFrame(raw))(fs.readFileSync(screenFile, 'utf-8'))
      : WAITING_PAGE;

    if (html.includes('</body>')) {
      html = html.replace('</body>', helperInjection + '\n</body>');
    } else {
      html += helperInjection;
    }

    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(html);
  } else if (req.method === 'GET' && req.url.startsWith('/files/')) {
    const fileName = req.url.slice(7); // strip '/files/'
    const filePath = path.join(SCREEN_DIR, path.basename(fileName));
    if (!fs.existsSync(filePath)) {
      res.writeHead(404);
      res.end('Not found');
      return;
    }
    const ext = path.extname(filePath).toLowerCase();
    const contentType = MIME_TYPES[ext] || 'application/octet-stream';
    res.writeHead(200, { 'Content-Type': contentType });
    res.end(fs.readFileSync(filePath));
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
}
```

- [ ] **Step 4: WebSocket connection handling を追加する**

```js
const clients = new Set();

function handleUpgrade(req, socket) {
  const key = req.headers['sec-websocket-key'];
  if (!key) { socket.destroy(); return; }

  const accept = computeAcceptKey(key);
  socket.write(
    'HTTP/1.1 101 Switching Protocols\r\n' +
    'Upgrade: websocket\r\n' +
    'Connection: Upgrade\r\n' +
    'Sec-WebSocket-Accept: ' + accept + '\r\n\r\n'
  );

  let buffer = Buffer.alloc(0);
  clients.add(socket);

  socket.on('data', (chunk) => {
    buffer = Buffer.concat([buffer, chunk]);
    while (buffer.length > 0) {
      let result;
      try {
        result = decodeFrame(buffer);
      } catch (e) {
        socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
        clients.delete(socket);
        return;
      }
      if (!result) break;
      buffer = buffer.slice(result.bytesConsumed);

      switch (result.opcode) {
        case OPCODES.TEXT:
          handleMessage(result.payload.toString());
          break;
        case OPCODES.CLOSE:
          socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
          clients.delete(socket);
          return;
        case OPCODES.PING:
          socket.write(encodeFrame(OPCODES.PONG, result.payload));
          break;
        case OPCODES.PONG:
          break;
        default:
          // Unsupported opcode — close with 1003
          const closeBuf = Buffer.alloc(2);
          closeBuf.writeUInt16BE(1003);
          socket.end(encodeFrame(OPCODES.CLOSE, closeBuf));
          clients.delete(socket);
          return;
      }
    }
  });

  socket.on('close', () => clients.delete(socket));
  socket.on('error', () => clients.delete(socket));
}

function handleMessage(text) {
  let event;
  try {
    event = JSON.parse(text);
  } catch (e) {
    console.error('Failed to parse WebSocket message:', e.message);
    return;
  }
  console.log(JSON.stringify({ source: 'user-event', ...event }));
  if (event.choice) {
    const eventsFile = path.join(SCREEN_DIR, '.events');
    fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
  }
}

function broadcast(msg) {
  const frame = encodeFrame(OPCODES.TEXT, Buffer.from(JSON.stringify(msg)));
  for (const socket of clients) {
    try { socket.write(frame); } catch (e) { clients.delete(socket); }
  }
}
```

- [ ] **Step 5: debounce timer map を追加する**

```js
const debounceTimers = new Map();
```

file watching logic は `startServer`（Step 6）内に inline で置く。watcher の lifecycle を server lifecycle とまとめ、spec にある `error` handler も含めるためである。

- [ ] **Step 6: startServer 関数と conditional main を追加する**

`frameTemplate` と `helperInjection` はすでに module scope（Step 2）にある。`startServer` は screen dir を作成し、HTTP server と watcher を起動し、startup info を log するだけでよい。

```js
function startServer() {
  if (!fs.existsSync(SCREEN_DIR)) fs.mkdirSync(SCREEN_DIR, { recursive: true });

  const server = http.createServer(handleRequest);
  server.on('upgrade', handleUpgrade);

  const watcher = fs.watch(SCREEN_DIR, (eventType, filename) => {
    if (!filename || !filename.endsWith('.html')) return;
    if (debounceTimers.has(filename)) clearTimeout(debounceTimers.get(filename));
    debounceTimers.set(filename, setTimeout(() => {
      debounceTimers.delete(filename);
      const filePath = path.join(SCREEN_DIR, filename);
      if (eventType === 'rename' && fs.existsSync(filePath)) {
        const eventsFile = path.join(SCREEN_DIR, '.events');
        if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);
        console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      } else if (eventType === 'change') {
        console.log(JSON.stringify({ type: 'screen-updated', file: filePath }));
      }
      broadcast({ type: 'reload' });
    }, 100));
  });
  watcher.on('error', (err) => console.error('fs.watch error:', err.message));

  server.listen(PORT, HOST, () => {
    const info = JSON.stringify({
      type: 'server-started', port: Number(PORT), host: HOST,
      url_host: URL_HOST, url: 'http://' + URL_HOST + ':' + PORT,
      screen_dir: SCREEN_DIR
    });
    console.log(info);
    fs.writeFileSync(path.join(SCREEN_DIR, '.server-info'), info + '\n');
  });
}

if (require.main === module) {
  startServer();
}
```

- [ ] **Step 7: integration tests を実行する**

test directory にはすでに `ws` を dependency に持つ `package.json` がある。必要なら install してから tests を実行する。

Run: `cd tests/brainstorm-server && npm install && node server.test.js`
Expected: すべての tests が pass する

- [ ] **Step 8: Commit**

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add HTTP server, WebSocket handling, and file watching to server.js"
```

---

## Chunk 3: Swap and Cleanup

### Task 3: start-server.sh を更新し、古いファイルを削除する

**Files:**
- Modify: `skills/brainstorming/scripts/start-server.sh:94,100`
- Modify: `.gitignore:6`
- Delete: `skills/brainstorming/scripts/index.js`
- Delete: `skills/brainstorming/scripts/package.json`
- Delete: `skills/brainstorming/scripts/package-lock.json`
- Delete: `skills/brainstorming/scripts/node_modules/`（directory 全体）

- [ ] **Step 1: start-server.sh を更新する — `index.js` を `server.js` に変更する**

変更するのは 2 行:

Line 94: `env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js`

Line 100: `nohup env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js > "$LOG_FILE" 2>&1 &`

- [ ] **Step 2: node_modules 用の gitignore 例外を削除する**

`.gitignore` から 6 行目 `!skills/brainstorming/scripts/node_modules/` を削除する。

- [ ] **Step 3: 古いファイルを削除する**

```bash
git rm skills/brainstorming/scripts/index.js
git rm skills/brainstorming/scripts/package.json
git rm skills/brainstorming/scripts/package-lock.json
git rm -r skills/brainstorming/scripts/node_modules/
```

- [ ] **Step 4: 両方の test suite を実行する**

Run: `cd tests/brainstorm-server && node ws-protocol.test.js && node server.test.js`
Expected: すべての tests が pass する

- [ ] **Step 5: Commit**

```bash
git add skills/brainstorming/scripts/ .gitignore
git commit -m "Remove vendored node_modules, swap to zero-dep server.js"
```

### Task 4: 手動 smoke test

- [ ] **Step 1: server を手動で起動する**

```bash
cd skills/brainstorming/scripts
BRAINSTORM_DIR=/tmp/brainstorm-smoke BRAINSTORM_PORT=9876 node server.js
```

Expected: port 9876 を含む `server-started` JSON が表示される

- [ ] **Step 2: browser で http://localhost:9876 を開く**

Expected: "Waiting for Claude to push a screen..." を含む waiting page が表示される

- [ ] **Step 3: screen directory に HTML ファイルを書き込む**

```bash
echo '<h2>Hello from smoke test</h2>' > /tmp/brainstorm-smoke/test.html
```

Expected: browser が reload し、frame template で wrap された "Hello from smoke test" が表示される

- [ ] **Step 4: WebSocket が動作することを確認する — browser console を確認する**

browser dev tools を開く。WebSocket connection は connected として表示されるべきであり、console に error は出ない。frame template の status indicator には "Connected" と表示されるはずである。

- [ ] **Step 5: Ctrl-C で server を停止し、後片付けする**

```bash
rm -rf /tmp/brainstorm-smoke
```

