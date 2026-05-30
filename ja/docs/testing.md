# Superpowers スキルのテスト

このドキュメントでは、Superpowers スキルのテスト方法、特に `subagent-driven-development` のような複雑なスキル向けの統合テストについて説明します。

## 概要

サブエージェント、ワークフロー、複雑な相互作用を含むスキルのテストでは、実際の Claude Code セッションをヘッドレスモードで実行し、セッショントランスクリプトを通じてその挙動を検証する必要があります。

## テスト構成

```
tests/
├── claude-code/
│   ├── test-helpers.sh                    # 共通のテストユーティリティ
│   ├── test-subagent-driven-development-integration.sh
│   ├── analyze-token-usage.py             # トークン分析ツール
│   └── run-skill-tests.sh                 # テストランナー（存在する場合）
```

## テストの実行

### 統合テスト

統合テストでは、実際のスキルを使って本物の Claude Code セッションを実行します:

```bash
# subagent-driven-development の統合テストを実行
cd tests/claude-code
./test-subagent-driven-development-integration.sh
```

**注:** 統合テストでは複数のサブエージェントを含む実際の実装プランを実行するため、10〜30 分かかることがあります。

### 要件

- **superpowers プラグインディレクトリ**から実行する必要があります（一時ディレクトリからではありません）
- Claude Code がインストールされ、`claude` コマンドとして利用可能であること
- ローカル開発マーケットプレイスが有効であること: `~/.claude/settings.json` に `"superpowers@superpowers-dev": true` を設定

## 統合テスト: subagent-driven-development

### 何をテストするか

統合テストでは、`subagent-driven-development` スキルが次を正しく行うことを検証します:

1. **Plan Loading**: 最初に一度だけプランを読み込む
2. **Full Task Text**: 完全なタスク説明をサブエージェントに渡す（ファイルを読ませない）
3. **Self-Review**: サブエージェントが報告前にセルフレビューを行うことを保証する
4. **Review Order**: コード品質レビューの前に仕様準拠レビューを実行する
5. **Review Loops**: 問題が見つかったときにレビューループを使う
6. **Independent Verification**: 仕様レビュアーが実装者の報告をうのみにせず、独立してコードを読む

### 仕組み

1. **Setup**: 最小限の実装プランを持つ一時的な Node.js プロジェクトを作成します
2. **Execution**: スキル付きで Claude Code をヘッドレスモードで実行します
3. **Verification**: セッショントランスクリプト（`.jsonl` ファイル）を解析して次を検証します:
   - Skill ツールが呼び出された
   - サブエージェントが起動された（Task ツール）
   - 追跡に TodoWrite が使われた
   - 実装ファイルが作成された
   - テストが通る
   - Git コミットが適切なワークフローを示している
4. **Token Analysis**: サブエージェントごとのトークン使用量の内訳を表示します

### テスト出力

```
========================================
 Integration Test: subagent-driven-development
========================================

Test project: /tmp/tmp.xyz123

=== Verification Tests ===

Test 1: Skill tool invoked...
  [PASS] subagent-driven-development skill was invoked

Test 2: Subagents dispatched...
  [PASS] 7 subagents dispatched

Test 3: Task tracking...
  [PASS] TodoWrite used 5 time(s)

Test 6: Implementation verification...
  [PASS] src/math.js created
  [PASS] add function exists
  [PASS] multiply function exists
  [PASS] test/math.test.js created
  [PASS] Tests pass

Test 7: Git commit history...
  [PASS] Multiple commits created (3 total)

Test 8: No extra features added...
  [PASS] No extra features added

=========================================
 Token Usage Analysis
=========================================

Usage Breakdown:
----------------------------------------------------------------------------------------------------
Agent           Description                          Msgs      Input     Output      Cache     Cost
----------------------------------------------------------------------------------------------------
main            Main session (coordinator)             34         27      3,996  1,213,703 $   4.09
3380c209        implementing Task 1: Create Add Function     1          2        787     24,989 $   0.09
34b00fde        implementing Task 2: Create Multiply Function     1          4        644     25,114 $   0.09
3801a732        reviewing whether an implementation matches...   1          5        703     25,742 $   0.09
4c142934        doing a final code review...                    1          6        854     25,319 $   0.09
5f017a42        a code reviewer. Review Task 2...               1          6        504     22,949 $   0.08
a6b7fbe4        a code reviewer. Review Task 1...               1          6        515     22,534 $   0.08
f15837c0        reviewing whether an implementation matches...   1          6        416     22,485 $   0.07
----------------------------------------------------------------------------------------------------

TOTALS:
  Total messages:         41
  Input tokens:           62
  Output tokens:          8,419
  Cache creation tokens:  132,742
  Cache read tokens:      1,382,835

  Total input (incl cache): 1,515,639
  Total tokens:             1,524,058

  Estimated cost: $4.67
  (at $3/$15 per M tokens for input/output)

========================================
 Test Summary
========================================

STATUS: PASSED
```

## トークン分析ツール

### 使い方

任意の Claude Code セッションのトークン使用量を分析します:

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

### セッションファイルの見つけ方

セッショントランスクリプトは `~/.claude/projects/` に、作業ディレクトリのパスをエンコードした形で保存されます:

```bash
# /Users/yourname/Documents/GitHub/superpowers/superpowers の例
SESSION_DIR="$HOME/.claude/projects/-Users-yourname-Documents-GitHub-superpowers-superpowers"

# 最近のセッションを探す
ls -lt "$SESSION_DIR"/*.jsonl | head -5
```

### 表示内容

- **メインセッションの使用量**: コーディネーター（あなた、またはメインの Claude インスタンス）のトークン使用量
- **サブエージェントごとの内訳**: 各 Task 呼び出しについて以下を表示します:
  - Agent ID
  - 説明（プロンプトから抽出）
  - メッセージ数
  - 入出力トークン
  - キャッシュ使用量
  - 推定コスト
- **合計**: 全体のトークン使用量と推定コスト

### 出力の見方

- **High cache reads**: 良い兆候です。プロンプトキャッシュが機能していることを意味します
- **High input tokens on main**: 想定どおりです。コーディネーターは完全なコンテキストを持っています
- **Similar costs per subagent**: 想定どおりです。各サブエージェントに同程度のタスク複雑度が割り当てられています
- **Cost per task**: タスクごとの典型的な範囲は、内容に応じてサブエージェントあたり $0.05〜$0.15 です

## トラブルシューティング

### スキルが読み込まれない

**問題**: ヘッドレステスト実行時にスキルが見つからない

**解決策**:
1. 必ず superpowers ディレクトリから実行してください: `cd /path/to/superpowers && tests/...`
2. `~/.claude/settings.json` の `enabledPlugins` に `"superpowers@superpowers-dev": true` があることを確認してください
3. スキルが `skills/` ディレクトリに存在することを確認してください

### 権限エラー

**問題**: Claude がファイル書き込みやディレクトリアクセスをブロックされる

**解決策**:
1. `--permission-mode bypassPermissions` フラグを使います
2. `--add-dir /path/to/temp/dir` を使ってテストディレクトリへのアクセス権を付与します
3. テストディレクトリのファイル権限を確認します

### テストのタイムアウト

**問題**: テストに時間がかかりすぎてタイムアウトする

**解決策**:
1. タイムアウトを延長します: `timeout 1800 claude ...`（30 分）
2. スキルロジックに無限ループがないか確認します
3. サブエージェントのタスク複雑度を見直します

### セッションファイルが見つからない

**問題**: テスト実行後にセッショントランスクリプトが見つからない

**解決策**:
1. `~/.claude/projects/` 内の正しいプロジェクトディレクトリを確認します
2. `find ~/.claude/projects -name "*.jsonl" -mmin -60` を使って最近のセッションを探します
3. テストが実際に実行されたことを確認します（テスト出力のエラーを確認）

## 新しい統合テストを書く

### テンプレート

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# テストプロジェクトを作成
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT

# テストファイルをセットアップ...
cd "$TEST_PROJECT"

# スキル付きで Claude を実行
PROMPT="Your test prompt here"
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT" \
  --allowed-tools=all \
  --add-dir "$TEST_PROJECT" \
  --permission-mode bypassPermissions \
  2>&1 | tee output.txt

# セッションを見つけて分析
WORKING_DIR_ESCAPED=$(echo "$SCRIPT_DIR/../.." | sed 's/\//-/g' | sed 's/^-//')
SESSION_DIR="$HOME/.claude/projects/$WORKING_DIR_ESCAPED"
SESSION_FILE=$(find "$SESSION_DIR" -name "*.jsonl" -type f -mmin -60 | sort -r | head -1)

# セッショントランスクリプトを解析して挙動を検証
if grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"; then
    echo "[PASS] Skill was invoked"
fi

# トークン分析を表示
python3 "$SCRIPT_DIR/analyze-token-usage.py" "$SESSION_FILE"
```

### ベストプラクティス

1. **Always cleanup**: trap を使って一時ディレクトリをクリーンアップする
2. **Parse transcripts**: ユーザー向け出力を grep せず、`.jsonl` セッションファイルを解析する
3. **Grant permissions**: `--permission-mode bypassPermissions` と `--add-dir` を使う
4. **Run from plugin dir**: スキルは superpowers ディレクトリから実行したときだけ読み込まれる
5. **Show token usage**: コストを見えるようにするため、必ずトークン分析を含める
6. **Test real behavior**: 実際にファイルが作成されたか、テストが通るか、コミットが作成されたかを検証する

## セッショントランスクリプト形式

セッショントランスクリプトは JSONL（JSON Lines）形式のファイルで、各行がメッセージまたはツール結果を表す JSON オブジェクトになっています。

### 主なフィールド

```json
{
  "type": "assistant",
  "message": {
    "content": [...],
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703
    }
  }
}
```

### ツール結果

```json
{
  "type": "user",
  "toolUseResult": {
    "agentId": "3380c209",
    "usage": {
      "input_tokens": 2,
      "output_tokens": 787,
      "cache_read_input_tokens": 24989
    },
    "prompt": "You are implementing Task 1...",
    "content": [{"type": "text", "text": "..."}]
  }
}
```

`agentId` フィールドはサブエージェントセッションに対応しており、`usage` フィールドにはそのサブエージェント呼び出し固有のトークン使用量が含まれています。
