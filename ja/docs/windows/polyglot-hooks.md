# Claude Code 向けクロスプラットフォーム・ポリグロットフック

Claude Code プラグインには、Windows、macOS、Linux で動作するフックが必要です。このドキュメントでは、それを可能にするポリグロットラッパー技法を説明します。

## 問題

Claude Code はフックコマンドをシステムのデフォルトシェル経由で実行します:
- **Windows**: CMD.exe
- **macOS/Linux**: bash または sh

これにより、いくつかの課題が生じます:

1. **Script execution**: Windows の CMD は `.sh` ファイルを直接実行できず、テキストエディタで開こうとします
2. **Path format**: Windows はバックスラッシュ（`C:\path`）、Unix はスラッシュ（`/path`）を使います
3. **Environment variables**: `$VAR` 構文は CMD では動作しません
4. **No `bash` in PATH**: Git Bash がインストールされていても、CMD 実行時には `bash` が PATH に入っていません

## 解決策: ポリグロット `.cmd` ラッパー

ポリグロットスクリプトとは、複数の言語で同時に有効な構文を持つスクリプトです。私たちのラッパーは CMD と bash の両方で有効です:

```cmd
: << 'CMDBLOCK'
@echo off
"C:\Program Files\Git\bin\bash.exe" -l -c "\"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")/hooks/session-start.sh\""
exit /b
CMDBLOCK

# Unix シェルはここから実行される
"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
```

### 仕組み

#### Windows (CMD.exe) 上

1. `: << 'CMDBLOCK'` - CMD は `:` をラベル（`label:` のようなもの）として見なし、`<< 'CMDBLOCK'` を無視します
2. `@echo off` - コマンドのエコー表示を抑制します
3. `bash.exe` コマンドは次の指定で実行されます:
   - `-l`（ログインシェル）で、Unix ユーティリティを含む適切な PATH を取得します
   - `cygpath -u` が Windows パスを Unix 形式に変換します（`C:\foo` → `/c/foo`）
4. `exit /b` - バッチスクリプトを終了し、CMD の処理をここで止めます
5. `CMDBLOCK` 以降は CMD では決して到達しません

#### Unix (bash/sh) 上

1. `: << 'CMDBLOCK'` - `:` は no-op で、`<< 'CMDBLOCK'` はヒアドキュメントを開始します
2. `CMDBLOCK` までの内容はヒアドキュメントとして消費され（無視され）ます
3. `# Unix シェルはここから実行される` - コメント
4. スクリプトは Unix パスで直接実行されます

## ファイル構成

```
hooks/
├── hooks.json           # .cmd ラッパーを指す
├── session-start.cmd    # ポリグロットラッパー（クロスプラットフォームのエントリーポイント）
└── session-start.sh     # 実際のフックロジック（bash スクリプト）
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.cmd\""
          }
        ]
      }
    ]
  }
}
```

注: `${CLAUDE_PLUGIN_ROOT}` に Windows 上の空白を含むパス（例: `C:\Program Files\...`）が入る可能性があるため、パスは必ず引用する必要があります。

## 要件

### Windows
- **Git for Windows** がインストールされている必要があります（`bash.exe` と `cygpath` を提供します）
- デフォルトのインストールパス: `C:\Program Files\Git\bin\bash.exe`
- Git が別の場所にインストールされている場合は、ラッパーの修正が必要です

### Unix (macOS/Linux)
- 標準の bash または sh シェル
- `.cmd` ファイルに実行権限が必要です（`chmod +x`）

## クロスプラットフォームなフックスクリプトを書く

実際のフックロジックは `.sh` ファイルに書きます。Windows で（Git Bash 経由で）確実に動かすには:

### 推奨
- 可能な限り bash のビルトインだけを使う
- バッククォートの代わりに `$(command)` を使う
- すべての変数展開を引用する: `"$VAR"`
- 出力には `printf` またはヒアドキュメントを使う

### 避けること
- PATH にない可能性がある外部コマンド（sed、awk、grep）
- どうしても必要なら Git Bash で利用できますが、PATH が適切に設定されていることを確認してください（`bash -l` を使います）

### 例: sed/awk を使わない JSON エスケープ

次の代わりに:
```bash
escaped=$(echo "$content" | sed 's/\\/\\\\/g' | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
```

純粋な bash を使います:
```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 再利用可能なラッパーパターン

複数のフックを持つプラグインでは、スクリプト名を引数として受け取る汎用ラッパーを作成できます:

### run-hook.cmd
```cmd
: << 'CMDBLOCK'
@echo off
set "SCRIPT_DIR=%~dp0"
set "SCRIPT_NAME=%~1"
"C:\Program Files\Git\bin\bash.exe" -l -c "cd \"$(cygpath -u \"%SCRIPT_DIR%\")\" && \"./%SCRIPT_NAME%\""
exit /b
CMDBLOCK

# Unix シェルはここから実行される
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_NAME="$1"
shift
"${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### 再利用可能ラッパーを使う hooks.json
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

## トラブルシューティング

### "bash is not recognized"
CMD が bash を見つけられません。ラッパーでは完全パス `C:\Program Files\Git\bin\bash.exe` を使っています。Git が別の場所にインストールされている場合は、そのパスに更新してください。

### "cygpath: command not found" または "dirname: command not found"
bash がログインシェルとして実行されていません。`-l` フラグが使われていることを確認してください。

### パスに奇妙な `\/` が入る
`${CLAUDE_PLUGIN_ROOT}` が末尾にバックスラッシュを持つ Windows パスへ展開されたあと、`/hooks/...` が連結されています。パス全体を `cygpath` で変換してください。

### スクリプトが実行されずテキストエディタで開かれる
`hooks.json` が `.sh` ファイルを直接指しています。.cmd ラッパーを指すようにしてください。

### ターミナルでは動くがフックでは動かない
Claude Code はフックを異なる方法で実行することがあります。フック環境をシミュレートしてテストしてください:
```powershell
$env:CLAUDE_PLUGIN_ROOT = "C:\path\to\plugin"
cmd /c "C:\path\to\plugin\hooks\session-start.cmd"
```

## 関連 Issues

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) - .sh scripts open in editor on Windows
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) - Hooks don't work on Windows
- [anthropics/claude-code#6023](https://github.com/anthropics/claude-code/issues/6023) - CLAUDE_PROJECT_DIR not found
