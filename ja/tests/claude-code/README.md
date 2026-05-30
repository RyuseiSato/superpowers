# Claude Code Skills テスト

Claude Code CLI を使って superpowers skills を検証する自動テストです。

## 概要

このテストスイートは、skills が正しく読み込まれ、Claude が期待どおりにそれに従うことを検証します。テストは Claude Code を headless モード（`claude -p`）で実行し、その振る舞いを確認します。

## 要件

- Claude Code CLI がインストールされ、PATH に入っていること（`claude --version` が動作すること）
- ローカルの superpowers plugin がインストールされていること（インストール方法はメイン README を参照）

## テストの実行

### すべての高速テストを実行（推奨）:
```bash
./run-skill-tests.sh
```

### 統合テストを実行（低速、10〜30 分）:
```bash
./run-skill-tests.sh --integration
```

### 特定のテストを実行:
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 詳細出力付きで実行:
```bash
./run-skill-tests.sh --verbose
```

### カスタム timeout を設定:
```bash
./run-skill-tests.sh --timeout 1800  # 統合テスト向けに 30 分
```

## テスト構成

### test-helpers.sh
skills テスト用の共通関数:
- `run_claude "prompt" [timeout]` - prompt を指定して Claude を実行
- `assert_contains output pattern name` - pattern が存在することを検証
- `assert_not_contains output pattern name` - pattern が存在しないことを検証
- `assert_count output pattern count name` - 正確な件数を検証
- `assert_order output pattern_a pattern_b name` - 順序を検証
- `create_test_project` - 一時的なテストディレクトリを作成
- `create_test_plan project_dir` - サンプルの plan ファイルを作成

### テストファイル

各テストファイルは以下を行います:
1. `test-helpers.sh` を source する
2. 特定の prompt で Claude Code を実行する
3. assertions を使って期待される振る舞いを検証する
4. 成功時は 0、失敗時は非 0 を返す

## テスト例

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# skill について Claude に尋ねる
output=$(run_claude "What does the my-skill skill do?" 30)

# 応答を検証
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 現在のテスト

### 高速テスト（デフォルトで実行）

#### test-subagent-driven-development.sh
skill の内容と要件をテストします（約 2 分）:
- skill の読み込みとアクセス可能性
- ワークフロー順序（code quality より前に spec compliance）
- self-review 要件が文書化されている
- plan の読み取り効率が文書化されている
- spec compliance reviewer の懐疑的な姿勢が文書化されている
- review loop が文書化されている
- task context の提供が文書化されている

### 統合テスト（`--integration` フラグを使用）

#### test-subagent-driven-development-integration.sh
完全なワークフロー実行テスト（約 10〜30 分）:
- Node.js セットアップ付きの実際のテスト project を作成
- 2 つの task を持つ implementation plan を作成
- subagent-driven-development を使って plan を実行
- 実際の振る舞いを検証:
  - plan は開始時に一度だけ読み込まれる（task ごとではない）
  - 完全な task テキストが subagent の prompt に渡される
  - subagent は報告前に self-review を行う
  - code quality より前に spec compliance review が行われる
  - spec reviewer が独立して code を読む
  - 動作する実装が生成される
  - テストが通る
  - 適切な git commit が作成される

**このテストで検証すること:**
- ワークフローが実際に end-to-end で機能すること
- 改善内容が実際に適用されていること
- subagent が skill に正しく従うこと
- 最終的な code が機能し、テストされていること

#### test-requesting-code-review.sh
code reviewer subagent の振る舞いテスト（約 5 分）:
- baseline commit を持つ小さな project を構築
- 2 つ目の commit で本物の bug を 2 件仕込む（SQL injection、平文 password の処理）
- requesting-code-review skill 経由で code reviewer を dispatch
- reviewer が仕込んだ bug を Critical/Important severity で指摘し、approve を拒否することを検証

**このテストで検証すること:**
- skill が実際に動作する code reviewer subagent を dispatch すること
- reviewer template により、明白な security bug を見つけられる reviewer が生成されること
- reviewer が sycophantic ではないこと — 仕込まれた Critical issue を含む diff を approve しないこと

## 新しいテストの追加

1. 新しいテストファイルを作成: `test-<skill-name>.sh`
2. test-helpers.sh を source する
3. `run_claude` と assertions を使ってテストを書く
4. `run-skill-tests.sh` のテスト一覧に追加する
5. 実行可能にする: `chmod +x test-<skill-name>.sh`

## Timeout に関する注意

- デフォルト timeout: テストごとに 5 分
- Claude Code は応答に時間がかかることがある
- 必要に応じて `--timeout` で調整する
- 長時間化を避けるため、テストは焦点を絞るべき

## 失敗したテストのデバッグ

`--verbose` を付けると、Claude の完全な出力を確認できます:
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

verbose なしでは、失敗時のみ出力が表示されます。

## CI/CD 統合

CI で実行するには:
```bash
# CI 環境向けに明示的な timeout を指定して実行
./run-skill-tests.sh --timeout 900

# 終了コード 0 = success、非 0 = failure
```

## Notes

- テストが検証するのは skill の*指示*であり、完全な実行ではありません
- 完全なワークフローテストは非常に低速になります
- 重要な skill 要件の検証に集中してください
- テストは決定的であるべきです
- 実装詳細のテストは避けてください
