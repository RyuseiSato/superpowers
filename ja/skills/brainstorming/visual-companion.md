# Visual Companion ガイド

モックアップ、図、選択肢を見せるためのブラウザベースの visual brainstorming companion。

## 使うタイミング

セッション単位ではなく、質問ごとに判断する。基準は、**読むより見たほうがユーザーに伝わるか** である。

**ブラウザを使う** のは、内容そのものが視覚的なとき:

- **UI モックアップ** — ワイヤーフレーム、レイアウト、ナビゲーション構造、コンポーネント設計
- **アーキテクチャ図** — システム構成要素、データフロー、関係マップ
- **並べて比較するビジュアル** — 2 つのレイアウト、2 つの配色、2 つのデザイン方向を比較する場合
- **デザインの磨き込み** — 見た目や雰囲気、余白、視覚的階層について問う場合
- **空間的な関係** — 状態機械、フローチャート、図として表したエンティティ関係

**terminal を使う** のは、内容がテキストまたは表形式のとき:

- **要件とスコープの質問** — 「X は何を意味するか？」「どの機能がスコープ内か？」
- **概念的な A/B/C の選択** — 言葉で説明されたアプローチから選ぶ場合
- **トレードオフ一覧** — 長所/短所、比較表
- **技術的判断** — API 設計、データモデリング、アーキテクチャ方針の選択
- **確認質問** — 答えが視覚的な好みではなく言葉になるもの全般

UI に関する質問だからといって、自動的に視覚的な質問になるわけではない。「どんな wizard が欲しいですか？」は概念的なので terminal を使う。「この wizard レイアウトのうち、どれがしっくりきますか？」は視覚的なのでブラウザを使う。

## 仕組み

サーバーは HTML ファイルが置かれるディレクトリを監視し、最新のものをブラウザに配信する。あなたは `screen_dir` に HTML コンテンツを書き、ユーザーはそれをブラウザで見て、クリックで選択肢を選べる。選択結果は `state_dir/events` に記録され、次のターンで読み取る。

**コンテンツ断片 vs 完全なドキュメント:** HTML ファイルが `<!DOCTYPE` または `<html` で始まる場合、サーバーはそれをそのまま配信する（helper script だけ注入する）。それ以外の場合は、サーバーが自動的に frame template でラップし、ヘッダー、CSS theme、選択インジケーター、対話用のインフラ一式を追加する。**デフォルトではコンテンツ断片を書くこと。** ページ全体を完全に制御したいときだけ、完全な document を書く。

## セッション開始

```bash
# Start server with persistence (mockups saved to project)
scripts/start-server.sh --project-dir /path/to/project

# Returns: {"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

応答から `screen_dir` と `state_dir` を保存する。ユーザーには URL を開くよう伝える。

**接続情報の見つけ方:** サーバーは起動時の JSON を `$STATE_DIR/server-info` に書き出す。バックグラウンド起動して stdout を取得していない場合は、そのファイルを読んで URL と port を取得する。`--project-dir` を使っている場合は、セッションディレクトリを `<project>/.superpowers/brainstorm/` で探す。

**注意:** モックアップを `.superpowers/brainstorm/` に保持し、サーバー再起動後も残るよう、`--project-dir` にはプロジェクトルートを渡すこと。これを付けないとファイルは `/tmp` に置かれ、掃除される。まだならユーザーに `.superpowers/` を `.gitignore` に追加するよう促す。

**プラットフォームごとのサーバー起動:**

**Claude Code (macOS / Linux):**
```bash
# Default mode works — the script backgrounds the server itself
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows):**
```bash
# Windows auto-detects and uses foreground mode, which blocks the tool call.
# Use run_in_background: true on the Bash tool call so the server survives
# across conversation turns.
scripts/start-server.sh --project-dir /path/to/project
```
これを Bash tool 経由で呼ぶ場合は、`run_in_background: true` を設定する。次のターンで `$STATE_DIR/server-info` を読めば URL と port を取得できる。

**Codex:**
```bash
# Codex reaps background processes. The script auto-detects CODEX_CI and
# switches to foreground mode. Run it normally — no extra flags needed.
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI:**
```bash
# Use --foreground and set is_background: true on your shell tool call
# so the process survives across turns
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**その他の環境:** サーバーは会話ターンをまたいでバックグラウンドで動き続ける必要がある。環境が detached process を回収するなら、`--foreground` を使い、そのプラットフォームのバックグラウンド実行手段でコマンドを起動する。

ブラウザから URL にアクセスできない場合（リモート環境やコンテナ環境でよくある）は、ループバック以外の host に bind する:

```bash
scripts/start-server.sh   --project-dir /path/to/project   --host 0.0.0.0   --url-host localhost
```

返される URL JSON に表示するホスト名を制御するには `--url-host` を使う。

## ループ

1. **サーバーが生きていることを確認し**, その後 **HTML を `screen_dir` の新しいファイルに書く**:
   - 各書き込み前に `$STATE_DIR/server-info` が存在することを確認する。存在しない（または `$STATE_DIR/server-stopped` が存在する）場合、サーバーは停止しているので、続ける前に `start-server.sh` で再起動する。サーバーは無操作が 30 分続くと自動終了する。
   - 意味のあるファイル名を使う: `platform.html`, `visual-style.html`, `layout.html`
   - **ファイル名は絶対に再利用しない** — 各画面ごとに新しいファイルにする
   - Write tool を使う — **cat/heredoc は絶対に使わない**（terminal にノイズが出る）
   - サーバーは自動的に最新ファイルを配信する

2. **ユーザーに何が表示されるか伝え、そのターンを終える:**
   - URL を毎回知らせる（最初だけではなく毎回）
   - 画面の内容を短く文章で要約する（例: 「ホームページのレイアウト案を 3 つ表示しています」）
   - terminal で返答するよう伝える: 「見てみて、どう思うか教えてください。必要ならクリックして選択肢を選んでください。」

3. **次のターンで** — ユーザーが terminal で返答したあと:
   - `$STATE_DIR/events` があれば読む — ここにはブラウザでの操作（クリック、選択）が JSON Lines で入っている
   - terminal 上のテキストと合わせて全体像を把握する
   - 主たるフィードバックは terminal メッセージであり、`state_dir/events` は構造化された操作データを補うもの

4. **繰り返すか先へ進む** — フィードバックで現在の画面を変える必要があるなら、新しいファイル（例: `layout-v2.html`）を書く。現在のステップが検証されるまでは次の質問に進まない。

5. **terminal に戻るときは unload する** — 次のステップでブラウザが不要な場合（例: 確認質問、トレードオフの議論）、待機画面を出して古い内容を消す:

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   これにより、会話が先へ進んでいるのに、ユーザーが解決済みの選択肢を見続けてしまうのを防げる。次の視覚的な質問が来たら、通常どおり新しいコンテンツファイルを出す。

6. 完了まで繰り返す。

## コンテンツ断片を書く

ページ内に入る内容だけを書く。サーバーが自動的に frame template でラップし、ヘッダー、theme CSS、選択インジケーター、対話用インフラ一式を追加する。

**最小例:**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

これだけでよい。`<html>` も CSS も `<script>` タグも不要。サーバーがそれらを提供する。

## 利用できる CSS クラス

frame template は、あなたのコンテンツ用に以下の CSS クラスを提供する:

### Options (A/B/C の選択)

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**複数選択:** コンテナに `data-multiselect` を追加すると、ユーザーが複数の選択肢を選べるようになる。クリックするたびに切り替わる。インジケーターバーには件数が表示される。

```html
<div class="options" data-multiselect>
  <!-- same option markup — users can select/deselect multiple -->
</div>
```

### Cards (ビジュアルデザイン)

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup container

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### Split view (左右並び)

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### Pros/Cons

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock elements (ワイヤーフレーム用の部品)

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### タイポグラフィとセクション

- `h2` — ページタイトル
- `h3` — セクション見出し
- `.subtitle` — タイトル下の補助テキスト
- `.section` — 下マージン付きのコンテンツブロック
- `.label` — 小さな大文字ラベルテキスト

## ブラウザイベントの形式

ユーザーがブラウザで選択肢をクリックすると、その操作は `$STATE_DIR/events` に記録される（1 行につき 1 つの JSON オブジェクト）。新しい画面を push すると、ファイルは自動的にクリアされる。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完全なイベント列を見ると、ユーザーがどう探索したかが分かる。最終的に決める前に複数の選択肢をクリックすることもある。最後の `choice` イベントが最終選択であることが多いが、クリックのパターン自体が、ためらいや好みを示しており、追加で尋ねる価値がある場合もある。

`$STATE_DIR/events` が存在しない場合、ユーザーはブラウザで操作していない。terminal のテキストだけを使う。

## デザインのヒント

- **問いに応じて忠実度を調整する** — レイアウトなら wireframe、磨き込みの問いなら polish
- **各ページで問いを説明する** — 「どのレイアウトがよりプロフェッショナルに感じますか？」であって、「1 つ選んで」だけではない
- **先に進む前に反復する** — フィードバックで現在の画面が変わるなら、新しい版を書く
- 1 画面あたり **選択肢は最大 2〜4 個**
- **重要なときは実データを使う** — たとえば写真ポートフォリオなら、プレースホルダーではなく実際の画像（Unsplash）を使う。プレースホルダーではデザイン上の問題が見えにくくなる。
- **モックアップはシンプルに保つ** — ピクセル単位のデザインではなく、レイアウトと構造に集中する

## ファイル命名

- 意味のある名前を使う: `platform.html`, `visual-style.html`, `layout.html`
- ファイル名は再利用しない — 各画面は必ず新しいファイルにする
- 反復時は `layout-v2.html`, `layout-v3.html` のようにバージョン接尾辞を付ける
- サーバーは更新時刻が最新のファイルを配信する

## クリーンアップ

```bash
scripts/stop-server.sh $SESSION_DIR
```

セッションで `--project-dir` を使っていた場合、モックアップファイルは後で参照できるよう `.superpowers/brainstorm/` に残る。停止時に削除されるのは `/tmp` セッションだけである。

## 参考

- Frame template（CSS の参考）: `scripts/frame-template.html`
- Helper script（client-side）: `scripts/helper.js`
