# Go Fractals CLI - 設計

## 概要

ASCII art fractal を生成する command-line tool です。設定可能な出力で 2 種類の fractal をサポートします。

## 使い方

```bash
# Sierpinski triangle
fractals sierpinski --size 32 --depth 5

# Mandelbrot set
fractals mandelbrot --width 80 --height 24 --iterations 100

# Custom character
fractals sierpinski --size 16 --char '#'

# Help
fractals --help
fractals sierpinski --help
```

## コマンド

### `sierpinski`

再帰的な分割を使って Sierpinski triangle を生成します。

Flags:
- `--size`（デフォルト: 32）- 文字数で表した triangle の底辺幅
- `--depth`（デフォルト: 5）- 再帰の深さ
- `--char`（デフォルト: `*`）- 塗りつぶし点に使う文字

出力: triangle を 1 行ずつ stdout に出力します。

### `mandelbrot`

Mandelbrot set を ASCII art として描画します。反復回数を文字に対応付けます。

Flags:
- `--width`（デフォルト: 80）- 出力幅（文字数）
- `--height`（デフォルト: 24）- 出力高さ（文字数）
- `--iterations`（デフォルト: 100）- 発散計算の最大反復回数
- `--char`（デフォルト: gradient）- 単一文字、または省略時は gradient `" .:-=+*#%@"`

出力: 矩形を stdout に出力します。

## アーキテクチャ

```
cmd/
  fractals/
    main.go           # エントリポイント、CLI セットアップ
internal/
  sierpinski/
    sierpinski.go     # アルゴリズム
    sierpinski_test.go
  mandelbrot/
    mandelbrot.go     # アルゴリズム
    mandelbrot_test.go
  cli/
    root.go           # ルート command、help
    sierpinski.go     # Sierpinski subcommand
    mandelbrot.go     # Mandelbrot subcommand
```

## 依存関係

- Go 1.21+
- CLI 用に `github.com/spf13/cobra`

## 受け入れ条件

1. `fractals --help` で使い方が表示される
2. `fractals sierpinski` でそれらしい triangle が出力される
3. `fractals mandelbrot` でそれらしい Mandelbrot set が出力される
4. `--size`、`--width`、`--height`、`--depth`、`--iterations` flags が動作する
5. `--char` で出力文字をカスタマイズできる
6. 不正な入力では分かりやすい error message が表示される
7. すべてのテストが通る
