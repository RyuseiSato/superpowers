# Visual Brainstorming Companion 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: この計画は superpowers:executing-plans を使ってタスクごとに実装してください。

**Goal:** Claude にブレインストーミングセッション向けのブラウザベースの visual companion を提供する - terminal conversation と並行して mockup、prototype、interactive choice を表示する。

**Architecture:** Claude が HTML を temp file に書き出す。ローカルの Node.js server がその file を監視し、自動挿入された helper library とともに配信する。user interaction は WebSocket 経由で server stdout に流れ、Claude はそれを background task output として確認する。

**Tech Stack:** Node.js, Express, ws (WebSocket), chokidar (file watching)

---

## Task 1: Server Foundation を作成する

**Files:**
- Create: `lib/brainstorm-server/index.js`
- Create: `lib/brainstorm-server/package.json`

**Step 1: package.json を作成する**

```json
{
  "name": "brainstorm-server",
  "version": "1.0.0",
  "description": "Visual brainstorming companion server for Claude Code",
  "main": "index.js",
  "dependencies": {
    "chokidar": "^3.5.3",
    "express": "^4.18.2",
    "ws": "^8.14.2"
  }
}
```

**Step 2: 起動可能な最小 server を作成する**

```javascript
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const chokidar = require('chokidar');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || 3333;
const SCREEN_FILE = process.env.BRAINSTORM_SCREEN || '/tmp/brainstorm/screen.html';
const SCREEN_DIR = path.dirname(SCREEN_FILE);

// Ensure screen directory exists
if (!fs.existsSync(SCREEN_DIR)) {
  fs.mkdirSync(SCREEN_DIR, { recursive: true });
}

// Create default screen if none exists
if (!fs.existsSync(SCREEN_FILE)) {
  fs.writeFileSync(SCREEN_FILE, `<!DOCTYPE html>
<html>
<head>
  <title>Brainstorm Companion</title>
  <style>
    body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
    h1 { color: #333; }
    p { color: #666; }
  </style>
</head>
<body>
  <h1>Brainstorm Companion</h1>
  <p>Waiting for Claude to push a screen...</p>
</body>
</html>`);
}

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// Track connected browsers for reload notifications
const clients = new Set();

wss.on('connection', (ws) => {
  clients.add(ws);
  ws.on('close', () => clients.delete(ws));

  ws.on('message', (data) => {
    // User interaction event - write to stdout for Claude
    const event = JSON.parse(data.toString());
    console.log(JSON.stringify({ type: 'user-event', ...event }));
  });
});

// Serve current screen with helper.js injected
app.get('/', (req, res) => {
  let html = fs.readFileSync(SCREEN_FILE, 'utf-8');

  // Inject helper script before </body>
  const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
  const injection = `<script>\n${helperScript}\n</script>`;

  if (html.includes('</body>')) {
    html = html.replace('</body>', `${injection}\n</body>`);
  } else {
    html += injection;
  }

  res.type('html').send(html);
});

// Watch for screen file changes
chokidar.watch(SCREEN_FILE).on('change', () => {
  console.log(JSON.stringify({ type: 'screen-updated', file: SCREEN_FILE }));
  // Notify all browsers to reload
  clients.forEach(ws => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'reload' }));
    }
  });
});

server.listen(PORT, '127.0.0.1', () => {
  console.log(JSON.stringify({ type: 'server-started', port: PORT, url: `http://localhost:${PORT}` }));
});
```

**Step 3: npm install を実行する**

Run: `cd lib/brainstorm-server && npm install`
Expected: dependencies がインストールされる

**Step 4: server が起動することをテストする**

Run: `cd lib/brainstorm-server && timeout 3 node index.js || true`
Expected: `server-started` と port 情報を含む JSON が表示される

**Step 5: Commit**

```bash
git add lib/brainstorm-server/
git commit -m "feat: add brainstorm server foundation"
```

---

## Task 2: Helper Library を作成する

**Files:**
- Create: `lib/brainstorm-server/helper.js`

**Step 1: event の auto-capture を備えた helper.js を作成する**

```javascript
(function() {
  const WS_URL = 'ws://' + window.location.host;
  let ws = null;
  let eventQueue = [];

  function connect() {
    ws = new WebSocket(WS_URL);

    ws.onopen = () => {
      // Send any queued events
      eventQueue.forEach(e => ws.send(JSON.stringify(e)));
      eventQueue = [];
    };

    ws.onmessage = (msg) => {
      const data = JSON.parse(msg.data);
      if (data.type === 'reload') {
        window.location.reload();
      }
    };

    ws.onclose = () => {
      // Reconnect after 1 second
      setTimeout(connect, 1000);
    };
  }

  function send(event) {
    event.timestamp = Date.now();
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(event));
    } else {
      eventQueue.push(event);
    }
  }

  // Auto-capture clicks on interactive elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('button, a, [data-choice], [role="button"], input[type="submit"]');
    if (!target) return;

    // Don't capture regular link navigation
    if (target.tagName === 'A' && !target.dataset.choice) return;

    e.preventDefault();

    send({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice || null,
      id: target.id || null,
      className: target.className || null
    });
  });

  // Auto-capture form submissions
  document.addEventListener('submit', (e) => {
    e.preventDefault();
    const form = e.target;
    const formData = new FormData(form);
    const data = {};
    formData.forEach((value, key) => { data[key] = value; });

    send({
      type: 'submit',
      formId: form.id || null,
      formName: form.name || null,
      data: data
    });
  });

  // Auto-capture input changes (debounced)
  let inputTimeout = null;
  document.addEventListener('input', (e) => {
    const target = e.target;
    if (!target.matches('input, textarea, select')) return;

    clearTimeout(inputTimeout);
    inputTimeout = setTimeout(() => {
      send({
        type: 'input',
        name: target.name || null,
        id: target.id || null,
        value: target.value,
        inputType: target.type || target.tagName.toLowerCase()
      });
    }, 500); // 500ms debounce
  });

  // Expose for explicit use if needed
  window.brainstorm = {
    send: send,
    choice: (value, metadata = {}) => send({ type: 'choice', value, ...metadata })
  };

  connect();
})();
```

**Step 2: helper.js が構文的に正しいことを確認する**

Run: `node -c lib/brainstorm-server/helper.js`
Expected: syntax error がない

**Step 3: Commit**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "feat: add browser helper library for event capture"
```

---

## Task 3: Server のテストを書く

**Files:**
- Create: `tests/brainstorm-server/server.test.js`
- Create: `tests/brainstorm-server/package.json`

**Step 1: test 用 package.json を作成する**

```json
{
  "name": "brainstorm-server-tests",
  "version": "1.0.0",
  "scripts": {
    "test": "node server.test.js"
  }
}
```

**Step 2: server tests を書く**

```javascript
const { spawn } = require('child_process');
const http = require('http');
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');
const assert = require('assert');

const SERVER_PATH = path.join(__dirname, '../../lib/brainstorm-server/index.js');
const TEST_PORT = 3334;
const TEST_SCREEN = '/tmp/brainstorm-test/screen.html';

// Clean up test directory
function cleanup() {
  if (fs.existsSync(path.dirname(TEST_SCREEN))) {
    fs.rmSync(path.dirname(TEST_SCREEN), { recursive: true });
  }
}

async function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function fetch(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve({ status: res.statusCode, body: data }));
    }).on('error', reject);
  });
}

async function runTests() {
  cleanup();

  // Start server
  const server = spawn('node', [SERVER_PATH], {
    env: { ...process.env, BRAINSTORM_PORT: TEST_PORT, BRAINSTORM_SCREEN: TEST_SCREEN }
  });

  let stdout = '';
  server.stdout.on('data', (data) => { stdout += data.toString(); });
  server.stderr.on('data', (data) => { console.error('Server stderr:', data.toString()); });

  await sleep(1000); // Wait for server to start

  try {
    // Test 1: Server starts and outputs JSON
    console.log('Test 1: Server startup message');
    assert(stdout.includes('server-started'), 'Should output server-started');
    assert(stdout.includes(TEST_PORT.toString()), 'Should include port');
    console.log('  PASS');

    // Test 2: GET / returns HTML with helper injected
    console.log('Test 2: Serves HTML with helper injected');
    const res = await fetch(`http://localhost:${TEST_PORT}/`);
    assert.strictEqual(res.status, 200);
    assert(res.body.includes('brainstorm'), 'Should include brainstorm content');
    assert(res.body.includes('WebSocket'), 'Should have helper.js injected');
    console.log('  PASS');

    // Test 3: WebSocket connection and event relay
    console.log('Test 3: WebSocket relays events to stdout');
    stdout = ''; // Reset stdout capture
    const ws = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws.on('open', resolve));

    ws.send(JSON.stringify({ type: 'click', text: 'Test Button' }));
    await sleep(100);

    assert(stdout.includes('user-event'), 'Should relay user events');
    assert(stdout.includes('Test Button'), 'Should include event data');
    ws.close();
    console.log('  PASS');

    // Test 4: File change triggers reload notification
    console.log('Test 4: File change notifies browsers');
    const ws2 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws2.on('open', resolve));

    let gotReload = false;
    ws2.on('message', (data) => {
      const msg = JSON.parse(data.toString());
      if (msg.type === 'reload') gotReload = true;
    });

    // Modify the screen file
    fs.writeFileSync(TEST_SCREEN, '<html><body>Updated</body></html>');
    await sleep(500);

    assert(gotReload, 'Should send reload message on file change');
    ws2.close();
    console.log('  PASS');

    console.log('\nAll tests passed!');

  } finally {
    server.kill();
    cleanup();
  }
}

runTests().catch(err => {
  console.error('Test failed:', err);
  process.exit(1);
});
```

**Step 3: tests を実行する**

Run: `cd tests/brainstorm-server && npm install ws && node server.test.js`
Expected: すべての tests が pass する

**Step 4: Commit**

```bash
git add tests/brainstorm-server/
git commit -m "test: add brainstorm server integration tests"
```

---

## Task 4: Brainstorming Skill に Visual Companion を追加する

**Files:**
- Modify: `skills/brainstorming/SKILL.md`
- Create: `skills/brainstorming/visual-companion.md` (supporting doc)

**Step 1: supporting documentation を作成する**

Create `skills/brainstorming/visual-companion.md`:

```markdown
# Visual Companion Reference

## Starting the Server

バックグラウンドジョブとして実行する:

```bash
node ${PLUGIN_ROOT}/lib/brainstorm-server/index.js
```

ユーザーにはこう伝える: "I've started a visual companion at http://localhost:3333 - open it in a browser."

## Pushing Screens

HTML を `/tmp/brainstorm/screen.html` に書き込む。server がこの file を監視し、browser を自動更新する。

## Reading User Responses

background task output で JSON event を確認する:

```json
{"type":"user-event","type":"click","text":"Option A","choice":"optionA","timestamp":1234567890}
{"type":"user-event","type":"submit","data":{"notes":"My feedback"},"timestamp":1234567891}
```

Event types:
- **click**: user が button または `data-choice` element を click した
- **submit**: user が form を送信した（すべての form data を含む）
- **input**: user が field に入力した（500ms debounce）

## HTML Patterns

### Choice Cards

```html
<div class="options">
  <button data-choice="optionA">
    <h3>Option A</h3>
    <p>Description</p>
  </button>
  <button data-choice="optionB">
    <h3>Option B</h3>
    <p>Description</p>
  </button>
</div>
```

### Interactive Mockup

```html
<div class="mockup">
  <header data-choice="header">App Header</header>
  <nav data-choice="nav">Navigation</nav>
  <main data-choice="main">Content</main>
</div>
```

### Form with Notes

```html
<form>
  <label>Priority: <input type="range" name="priority" min="1" max="5"></label>
  <textarea name="notes" placeholder="Additional thoughts..."></textarea>
  <button type="submit">Submit</button>
</form>
```

### Explicit JavaScript

```html
<button onclick="brainstorm.choice('custom', {extra: 'data'})">Custom</button>
```
```

**Step 2: brainstorming skill に visual companion section を追加する**

`skills/brainstorming/SKILL.md` の "Key Principles" の後に追加:

```markdown

## Visual Companion (Optional)

ブレインストーミングに visual element - UI mockup、wireframe、interactive prototype - が含まれる場合は、ブラウザベースの visual companion を使う。

**When to use:**
- 視覚比較が有効な UI/UX option を提示するとき
- wireframe や layout option を見せるとき
- 構造化された feedback（rating、form）を集めるとき
- click interaction を prototype するとき

**How it works:**
1. server を background job として起動する
2. user に http://localhost:3333 を開くよう伝える
3. HTML を `/tmp/brainstorm/screen.html` に書く（自動更新される）
4. user interaction は background task output で確認する

terminal は引き続き主要な conversation interface です。browser は visual aid です。

**Reference:** HTML pattern と API detail は、この skill directory の `visual-companion.md` を参照。
```

**Step 3: edit を検証する**

Run: `grep -A5 "Visual Companion" skills/brainstorming/SKILL.md`
Expected: 新しい section が表示される

**Step 4: Commit**

```bash
git add skills/brainstorming/
git commit -m "feat: add visual companion to brainstorming skill"
```

---

## Task 5: Plugin Ignore に Server を追加する（Optional Cleanup）

**Files:**
- `.gitignore` に lib/brainstorm-server 用の node_modules exclusion が必要か確認する

**Step 1: 現在の gitignore を確認する**

Run: `cat .gitignore 2>/dev/null || echo "No .gitignore"`

**Step 2: 必要なら node_modules を追加する**

まだ存在しない場合は、以下を追加する:
```
lib/brainstorm-server/node_modules/
```

**Step 3: 変更があれば Commit**

```bash
git add .gitignore
git commit -m "chore: ignore brainstorm-server node_modules"
```

---

## まとめ

すべての tasks を完了したら:

1. **Server** at `lib/brainstorm-server/` - HTML file を監視し、event を relay する Node.js server
2. **Helper library** auto-injected - click、form、input を capture する
3. **Tests** at `tests/brainstorm-server/` - server behavior を検証する
4. **Brainstorming skill** を更新 - visual companion section と参照用 doc `visual-companion.md` を追加

**使い方:**
1. server を background job として起動する: `node lib/brainstorm-server/index.js &`
2. user に `http://localhost:3333` を開くよう伝える
3. HTML を `/tmp/brainstorm/screen.html` に書く
4. task output で user event を確認する
