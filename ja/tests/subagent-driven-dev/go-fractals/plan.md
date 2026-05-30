# Go Fractals CLI - 実装計画

この計画は `superpowers:subagent-driven-development` skill を使って実行してください。

## Context

ASCII fractal を生成する CLI tool を構築します。完全な仕様は `design.md` を参照してください。

## Tasks

### Task 1: Project Setup

Go module とディレクトリ構成を作成します。

**実施内容:**
- module 名 `github.com/superpowers-test/fractals` で `go.mod` を初期化
- ディレクトリ構成を作成: `cmd/fractals/`, `internal/sierpinski/`, `internal/mandelbrot/`, `internal/cli/`
- `fractals cli` を出力する最小限の `cmd/fractals/main.go` を作成
- `github.com/spf13/cobra` 依存関係を追加

**確認:**
- `go build ./cmd/fractals` が成功する
- `./fractals` が `fractals cli` を表示する

---

### Task 2: CLI Framework with Help

help 出力を持つ Cobra root command をセットアップします。

**実施内容:**
- root command を持つ `internal/cli/root.go` を作成
- 利用可能な subcommands が表示されるよう help text を設定
- root command を `main.go` に接続

**確認:**
- `./fractals --help` で使い方が表示され、利用可能な command として `sierpinski` と `mandelbrot` が列挙される
- `./fractals`（引数なし）で help が表示される

---

### Task 3: Sierpinski Algorithm

Sierpinski triangle 生成アルゴリズムを実装します。

**実施内容:**
- `internal/sierpinski/sierpinski.go` を作成
- 行の配列を返す `Generate(size, depth int, char rune) []string` を実装
- 再帰的な midpoint subdivision algorithm を使用
- 以下をテストする `internal/sierpinski/sierpinski_test.go` を作成:
  - 小さな triangle（size=4, depth=2）が期待出力と一致
  - size=1 で単一文字を返す
  - depth=0 で塗りつぶされた triangle を返す

**確認:**
- `go test ./internal/sierpinski/...` が通る

---

### Task 4: Sierpinski CLI Integration

Sierpinski algorithm を CLI subcommand に接続します。

**実施内容:**
- `sierpinski` subcommand を持つ `internal/cli/sierpinski.go` を作成
- flags を追加: `--size`（デフォルト 32）、`--depth`（デフォルト 5）、`--char`（デフォルト `*`）
- `sierpinski.Generate()` を呼び、結果を stdout に出力

**確認:**
- `./fractals sierpinski` で triangle が出力される
- `./fractals sierpinski --size 16 --depth 3` でより小さな triangle が出力される
- `./fractals sierpinski --help` で flag の説明が表示される

---

### Task 5: Mandelbrot Algorithm

Mandelbrot set の ASCII renderer を実装します。

**実施内容:**
- `internal/mandelbrot/mandelbrot.go` を作成
- `Render(width, height, maxIter int, char string) []string` を実装
- 複素平面領域（実数 -2.5 〜 1.0、虚数 -1.0 〜 1.0）を出力サイズに対応付ける
- 反復回数を文字 gradient `" .:-=+*#%@"` に対応付ける（または指定があれば単一文字）
- 以下をテストする `internal/mandelbrot/mandelbrot_test.go` を作成:
  - 出力サイズが要求した width/height と一致する
  - 集合内の既知の点（0,0）が max-iteration 文字に対応する
  - 集合外の既知の点（2,0）が低反復回数の文字に対応する

**確認:**
- `go test ./internal/mandelbrot/...` が通る

---

### Task 6: Mandelbrot CLI Integration

Mandelbrot algorithm を CLI subcommand に接続します。

**実施内容:**
- `mandelbrot` subcommand を持つ `internal/cli/mandelbrot.go` を作成
- flags を追加: `--width`（デフォルト 80）、`--height`（デフォルト 24）、`--iterations`（デフォルト 100）、`--char`（デフォルト `""`）
- `mandelbrot.Render()` を呼び、結果を stdout に出力

**確認:**
- `./fractals mandelbrot` でそれらしい Mandelbrot set が出力される
- `./fractals mandelbrot --width 40 --height 12` でより小さな版が出力される
- `./fractals mandelbrot --help` で flag の説明が表示される

---

### Task 7: Character Set Configuration

`--char` flag が両方の command で一貫して動作することを確認します。

**実施内容:**
- Sierpinski の `--char` flag が文字を algorithm に渡すことを確認
- Mandelbrot では `--char` 指定時に gradient ではなく単一文字を使用
- custom character 出力のテストを追加

**確認:**
- `./fractals sierpinski --char '#'` で `#` が使われる
- `./fractals mandelbrot --char '.'` で塗りつぶし点に `.` が使われる
- テストが通る

---

### Task 8: Input Validation and Error Handling

不正な入力に対する validation と error handling を追加します。

**実施内容:**
- Sierpinski: size は > 0、depth は >= 0
- Mandelbrot: width/height は > 0、iterations は > 0
- 不正入力には分かりやすい error message を返す
- error case のテストを追加

**確認:**
- `./fractals sierpinski --size 0` で error を表示し、非 0 で終了する
- `./fractals mandelbrot --width -1` で error を表示し、非 0 で終了する
- error message が明確で分かりやすい

---

### Task 9: Integration Tests

CLI を呼び出す integration テストを追加します。

**実施内容:**
- `cmd/fractals/main_test.go` または `test/integration_test.go` を作成
- 両方の command について完全な CLI 呼び出しをテスト
- 出力形式と終了コードを検証
- error case が非 0 を返すことをテスト

**確認:**
- `go test ./...` が integration テストを含めてすべて通る

---

### Task 10: README

使い方と例を文書化します。

**実施内容:**
- 以下を含む `README.md` を作成:
  - tool の説明
  - Installation: `go install ./cmd/fractals`
  - 両 command の usage examples
  - 出力例（小さなサンプル）

**確認:**
- README が tool を正確に説明している
- README 内の例が実際に動作する
