# Visual Brainstorming Refactor 実装計画

> **エージェントワーカー向け:** 必須: この計画の実装には superpowers:subagent-driven-development（サブエージェントが利用可能な場合）または superpowers:executing-plans を使用すること。追跡には checkbox (`- [ ]`) syntax を使う。

**目標:** visual brainstorming を、ブロッキングな TUI feedback モデルから、ノンブロッキングな「Browser Displays, Terminal Commands」アーキテクチャへリファクタリングする。

**アーキテクチャ:** Browser は対話型 display になり、terminal は会話チャネルのままにする。server は user event を screen ごとの `.events` ファイルに書き出し、Claude は次の turn でそれを読む。これにより `wait-for-feedback.sh` とすべての `TaskOutput` blocking を排除する。

**技術スタック:** Node.js (Express, ws, chokidar), vanilla HTML/CSS/JS

**仕様:** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `lib/brainstorm-server/index.js` | Modify | Server: `.events` ファイル書き出しを追加し、新しい screen でクリアし、`wrapInFrame` を置き換える |
| `lib/brainstorm-server/frame-template.html` | Modify | Template: feedback footer を削除し、content placeholder + selection indicator を追加する |
| `lib/brainstorm-server/helper.js` | Modify | Client JS: send/feedback 関数を削除し、click capture + indicator 更新に絞る |
| `lib/brainstorm-server/wait-for-feedback.sh` | Delete | もう不要 |
| `skills/brainstorming/visual-companion.md` | Modify | Skill instructions: ループをノンブロッキングな flow に書き換える |
| `tests/brainstorm-server/server.test.js` | Modify | Tests: 新しい template 構造と helper.js API に合わせて更新する |

---

## Chunk 1: Server, Template, Client, Tests, Skill

### Task 1: `frame-template.html` を更新する

**Files:**
- Modify: `lib/brainstorm-server/frame-template.html`

- [ ] **Step 1: feedback footer の HTML を削除する**

feedback-footer div（227-233 行）を selection indicator bar に置き換える:

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above, then return to the terminal</span>
  </div>
```

また、`#claude-content` 内のデフォルト content（220-223 行）を content placeholder に置き換える:

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **Step 2: feedback footer の CSS を indicator bar の CSS に置き換える**

`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、および `.feedback-footer` 内の textarea/button style（82-112 行）を削除する。

indicator bar の CSS を追加する:

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **Step 3: template が描画されることを確認する**

template が引き続き読み込まれることを確認するため、test suite を実行する:
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: Tests 1-5 は引き続き通る。Tests 6-8 は失敗する可能性がある（想定内 — 古い構造を検証しているため）。

- [ ] **Step 4: Commit**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### Task 2: `index.js` を更新する — content injection と `.events` ファイル

**Files:**
- Modify: `lib/brainstorm-server/index.js`

- [ ] **Step 1: `.events` ファイル書き込みの failing test を書く**

`tests/brainstorm-server/server.test.js` の Test 4 の後あたりに、`choice` フィールド付きの WebSocket event を送信し、`.events` ファイルが書き込まれることを確認する新しい test を追加する:

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **Step 2: test を実行して失敗することを確認する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: 新しい test は FAIL する — まだ `.events` ファイルが存在しないため。

- [ ] **Step 3: 新しい screen で `.events` ファイルがクリアされる failing test を書く**

別の test を追加する:

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **Step 4: test を実行して失敗することを確認する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: 新しい test は FAIL する — screen push 時に `.events` がクリアされないため。

- [ ] **Step 5: `index.js` に `.events` ファイル書き込みを実装する**

WebSocket の `message` handler（`index.js` 74-77 行）の `console.log` の後に、次を追加する:

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

また、chokidar の `add` handler（104-111 行）に `.events` のクリアを追加する:

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **Step 6: `wrapInFrame` を comment placeholder injection に置き換える**

`wrapInFrame` 関数（`index.js` 27-32 行）を次に置き換える:

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **Step 7: すべての tests を実行する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: 新しい `.events` tests は PASS する。既存 test の一部は、古い assertion が残っているためまだ失敗する可能性がある（Task 4 で修正）。

- [ ] **Step 8: Commit**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### Task 3: `helper.js` を簡素化する

**Files:**
- Modify: `lib/brainstorm-server/helper.js`

- [ ] **Step 1: `sendToClaude` 関数を削除する**

`sendToClaude` 関数（92-106 行）を削除する — 関数本体と page takeover HTML の両方。

- [ ] **Step 2: `window.send` 関数を削除する**

`window.send` 関数（120-129 行）を削除する — 削除済みの Send button に紐づいていたもの。

- [ ] **Step 3: form submission と input change handler を削除する**

form submission handler（57-71 行）と input change handler（73-89 行）を、`inputTimeout` 変数ごと削除する。

- [ ] **Step 4: `pageshow` event listener を削除する**

以前追加した `pageshow` listener を削除する（もはやクリアすべき textarea がないため）。

- [ ] **Step 5: click handler を `[data-choice]` のみに絞る**

click handler（36-55 行）を、より限定した次のバージョンに置き換える:

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **Step 6: choice click 時に indicator bar を更新する処理を追加する**

click handler の `sendEvent` 呼び出しの後に、次を追加する:

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **Step 7: `window.brainstorm` API から `sendToClaude` を削除する**

`window.brainstorm` object（132-136 行）を更新して `sendToClaude` を取り除く:

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **Step 8: tests を実行する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **Step 9: Commit**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### Task 4: 新しい構造に合わせて tests を更新する

**Files:**
- Modify: `tests/brainstorm-server/server.test.js`

**Note:** 以下の行番号は _元の_ ファイルを基準にしている。Task 2 で新しい tests を前の方に挿入するため、実際の行番号はずれる。`console.log` のラベル（例: "Test 5:", "Test 6:"）で test を探すこと。

- [ ] **Step 1: Test 5（full document assertion）を更新する**

Test 5 の assertion `!fullRes.body.includes('feedback-footer')` を見つけ、次のように変更する: full document には indicator bar も含まれないべきである（そのまま提供されるため）。

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **Step 2: Test 6（fragment wrapping）を更新する**

125 行の `feedback-footer` assertion を indicator bar の assertion に置き換える:

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

さらに、content placeholder が置き換えられていることも確認する（fragment content は現れ、placeholder comment は残らない）:

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **Step 3: Test 7（helper.js API）を更新する**

140-142 行の assertion を、新しい API surface に合わせて更新する:

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **Step 4: Test 8（sendToClaude theming）を indicator bar test に置き換える**

Test 8（145-149 行）は `sendToClaude` がもう存在しないため置き換える。代わりに indicator bar を test する:

```javascript
    // Test 8: Indicator bar uses CSS variables (theme support)
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **Step 5: full test suite を実行する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: すべての tests が PASS する。

- [ ] **Step 6: Commit**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### Task 5: `wait-for-feedback.sh` を削除する

**Files:**
- Delete: `lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **Step 1: 他のファイルが `wait-for-feedback.sh` を import または参照していないことを確認する**

codebase を検索する:
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

Expected references: `visual-companion.md`（Task 6 で書き換える）と、おそらく release notes のみ（履歴なのでそのままでよい）。

- [ ] **Step 2: ファイルを削除する**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **Step 3: 何も壊れていないことを確認するため tests を実行する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: すべての tests が PASS する（このファイルを参照する test は存在しない）。

- [ ] **Step 4: Commit**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### Task 6: `visual-companion.md` を書き換える

**Files:**
- Modify: `skills/brainstorming/visual-companion.md`

- [ ] **Step 1: "How It Works" の説明（18 行目）を更新する**

feedback を "JSON として" 受け取るという文を、次に置き換える:

```markdown
The server watches a directory for HTML files and serves the newest one to the browser. You write HTML content, the user sees it in their browser and can click to select options. Selections are recorded to a `.events` file that you read on your next turn.
```

- [ ] **Step 2: fragment の説明（20 行目）を更新する**

frame template が提供するものの説明から "feedback footer" を削除する:

```markdown
**Content fragments vs full documents:** If your HTML file starts with `<!DOCTYPE` or `<html`, the server serves it as-is (just injects the helper script). Otherwise, the server automatically wraps your content in the frame template — adding the header, CSS theme, selection indicator, and all interactive infrastructure. **Write content fragments by default.** Only write full documents when you need complete control over the page.
```

- [ ] **Step 3: "The Loop" セクション（36-61 行）を全面的に書き換える**

"The Loop" セクション全体を次に置き換える:

```markdown
## The Loop

1. **Write HTML** to a new file in `screen_dir`:
   - Use semantic filenames: `platform.html`, `visual-style.html`, `layout.html`
   - **Never reuse filenames** — each screen gets a fresh file
   - Use Write tool — **never use cat/heredoc** (dumps noise into terminal)
   - Server automatically serves the newest file

2. **Tell user what to expect and end your turn:**
   - Remind them of the URL (every step, not just first)
   - Give a brief text summary of what's on screen (e.g., "Showing 3 layout options for the homepage")
   - Ask them to respond in the terminal: "Take a look and let me know what you think. Click to select an option if you'd like."

3. **On your next turn** — after the user responds in the terminal:
   - Read `$SCREEN_DIR/.events` if it exists — this contains the user's browser interactions (clicks, selections) as JSON lines
   - Merge with the user's terminal text to get the full picture
   - The terminal message is the primary feedback; `.events` provides structured interaction data

4. **Iterate or advance** — if feedback changes current screen, write a new file (e.g., `layout-v2.html`). Only move to the next question when the current step is validated.

5. Repeat until done.
```

- [ ] **Step 4: "User Feedback Format" セクション（165-174 行）を置き換える**

次の内容に置き換える:

```markdown
## Browser Events Format

When the user clicks options in the browser, their interactions are recorded to `$SCREEN_DIR/.events` (one JSON object per line). The file is cleared automatically when you push a new screen.

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

The full event stream shows the user's exploration path — they may click multiple options before settling. The last `choice` event is typically the final selection, but the pattern of clicks can reveal hesitation or preferences worth asking about.

If `.events` doesn't exist, the user didn't interact with the browser — use only their terminal text.
```

- [ ] **Step 5: "Writing Content Fragments" の説明（65 行目）を更新する**

"feedback footer" への言及を削除する:

```markdown
Write just the content that goes inside the page. The server wraps it in the frame template automatically (header, theme CSS, selection indicator, and all interactive infrastructure).
```

- [ ] **Step 6: Reference セクション（200-203 行）を更新する**

helper.js の "JS API" 説明を削除する — API は最小限になった。path 参照は残す:

```markdown
## Reference

- Frame template (CSS reference): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script (client-side): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **Step 7: Commit**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### Task 7: 最終確認

- [ ] **Step 1: full test suite を実行する**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
Expected: すべての tests が PASS する。

- [ ] **Step 2: 手動 smoke test**

server を手動で起動し、flow が end-to-end で動作することを確認する:

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

test fragment を書き、browser で開き、option をクリックし、`.events` ファイルが書かれることと indicator bar が更新されることを確認する。その後 server を停止する:

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **Step 3: 古い参照が残っていないことを確認する**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

Expected: release notes と spec/plan docs を除き、ヒットはない。

- [ ] **Step 4: 必要なら最終 commit を行う**

```bash
git status
# Review untracked/modified files, stage specific files as needed, commit if clean
```

