# Superpowers

Superpowers は、合成可能なスキル群と、それらを確実に使うための初期命令の上に構築された、コーディングエージェント向けの完全なソフトウェア開発メソドロジーです。

## クイックスタート

あなたのエージェントに Superpowers を与えましょう: [Claude Code](#claude-code), [Codex CLI](#codex-cli), [Codex App](#codex-app), [Factory Droid](#factory-droid), [Gemini CLI](#gemini-cli), [OpenCode](#opencode), [Cursor](#cursor), [GitHub Copilot CLI](#github-copilot-cli).

## 仕組み

これは、あなたがコーディングエージェントを起動した瞬間から始まります。何かを作ろうとしているとエージェントが認識すると、*いきなり*コードを書き始めるのではありません。代わりに一歩引いて、あなたが本当に何を達成しようとしているのかを尋ねます。

会話の中から仕様を引き出すと、それを実際に読んで理解できるよう、短いまとまりに分けてあなたに見せます。

設計をあなたが承認すると、エージェントは、やる気はあるがセンスがなく、判断力もプロジェクトの文脈理解もなく、テストも嫌がるジュニアエンジニアでも従えるほど明確な実装計画をまとめます。そこでは、真の red/green TDD、YAGNI (You Aren't Gonna Need It)、そして DRY を重視します。

次に、あなたが「go」と言えば、*subagent-driven-development* プロセスを開始します。各エンジニアリングタスクをエージェントに担当させ、その作業を検査・レビューしながら前に進みます。Claude なら、あなたと一緒に組み立てた計画から逸脱することなく、一度に数時間自律的に作業できるのも珍しくありません。

他にもいろいろありますが、これがシステムの中核です。そして、スキルは自動的に発動するので、あなたが特別なことをする必要はありません。あなたのコーディングエージェントは、ただ Superpowers を手に入れるだけです。


## スポンサーシップ

もし Superpowers が、あなたの収益につながるようなことに役立っていて、そうする気持ちがあるなら、ぜひ [私のオープンソース活動をスポンサー](https://github.com/sponsors/obra) することをご検討いただけると、とてもありがたいです。

ありがとうございます！

- Jesse


## インストール

インストール方法はハーネスごとに異なります。複数使っている場合は、それぞれに対して個別に Superpowers をインストールしてください。

### Claude Code

Superpowers は [公式 Claude プラグインマーケットプレイス](https://claude.com/plugins/superpowers) から利用できます。

#### 公式マーケットプレイス

- Anthropic の公式マーケットプレイスからプラグインをインストールします:

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers Marketplace

Superpowers marketplace では、Claude Code 向けに Superpowers と関連プラグインをいくつか提供しています。

- マーケットプレイスを登録します:

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- このマーケットプレイスからプラグインをインストールします:

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Codex CLI

Superpowers は [公式 Codex プラグインマーケットプレイス](https://github.com/openai/plugins) から利用できます。

- プラグイン検索インターフェースを開きます:

  ```bash
  /plugins
  ```

- Superpowers を検索します:

  ```bash
  superpowers
  ```

- `Install Plugin` を選択します。

### Codex App

Superpowers は [公式 Codex プラグインマーケットプレイス](https://github.com/openai/plugins) から利用できます。

- Codex app で、サイドバーの Plugins をクリックします。
- Coding セクションに `Superpowers` が表示されるはずです。
- Superpowers の横にある `+` をクリックし、画面の案内に従ってください。

### Factory Droid

- マーケットプレイスを登録します:

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- プラグインをインストールします:

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 拡張機能をインストールします:

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 後で更新するには:

  ```bash
  gemini extensions update superpowers
  ```

### OpenCode

OpenCode は独自のプラグインインストール方式を使います。別のハーネスですでに使っている場合でも、Superpowers を個別にインストールしてください。

- OpenCode に次のように伝えます:

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 詳細なドキュメント: [docs/README.opencode.md](docs/README.opencode.md)

### Cursor

- Cursor Agent チャットで、マーケットプレイスからインストールします:

  ```text
  /add-plugin superpowers
  ```

- または、プラグインマーケットプレイスで "superpowers" を検索してください。

### GitHub Copilot CLI

- マーケットプレイスを登録します:

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- プラグインをインストールします:

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

## 基本ワークフロー

1. **brainstorming** - コードを書く前に発動します。質問を通じて粗いアイデアを洗練し、代替案を検討し、設計を検証のために分割して提示します。設計ドキュメントを保存します。

2. **using-git-worktrees** - 設計承認後に発動します。新しいブランチ上に隔離された作業空間を作成し、プロジェクトのセットアップを実行し、テストのクリーンなベースラインを確認します。

3. **writing-plans** - 承認済み設計とともに発動します。作業を小さなタスク（各 2〜5 分）に分解します。すべてのタスクに、正確なファイルパス、完全なコード、検証手順が含まれます。

4. **subagent-driven-development** または **executing-plans** - 計画とともに発動します。タスクごとに新しいサブエージェントを割り当て、2 段階レビュー（仕様準拠、その後コード品質）を行うか、人間のチェックポイント付きでバッチ実行します。

5. **test-driven-development** - 実装中に発動します。RED-GREEN-REFACTOR を徹底します: 失敗するテストを書く、失敗を確認する、最小限のコードを書く、成功を確認する、コミットする。テストより先に書かれたコードは削除します。

6. **requesting-code-review** - タスクの合間に発動します。計画に照らしてレビューし、重大度ごとに問題を報告します。重大な問題は進行を止めます。

7. **finishing-a-development-branch** - タスク完了時に発動します。テストを確認し、選択肢（merge/PR/keep/discard）を提示し、worktree を片付けます。

**エージェントはどのタスクの前でも、関連するスキルがあるかを確認します。** これは提案ではなく、必須のワークフローです。

## 内容

### スキルライブラリ

**Testing**
- **test-driven-development** - RED-GREEN-REFACTOR サイクル（テストのアンチパターン集を含む）

**Debugging**
- **systematic-debugging** - 4 段階の根本原因プロセス（root-cause-tracing、defense-in-depth、condition-based-waiting の各テクニックを含む）
- **verification-before-completion** - 本当に修正されたことを確認する

**Collaboration** 
- **brainstorming** - ソクラテス式の設計洗練
- **writing-plans** - 詳細な実装計画
- **executing-plans** - チェックポイント付きバッチ実行
- **dispatching-parallel-agents** - 並行サブエージェントワークフロー
- **requesting-code-review** - 事前レビュー用チェックリスト
- **receiving-code-review** - フィードバックへの対応
- **using-git-worktrees** - 並行開発ブランチ
- **finishing-a-development-branch** - merge/PR 判断ワークフロー
- **subagent-driven-development** - 2 段階レビュー（仕様準拠、その後コード品質）による高速反復

**Meta**
- **writing-skills** - ベストプラクティスに従って新しいスキルを作る（テスト手法を含む）
- **using-superpowers** - スキルシステムの紹介

## 哲学

- **Test-Driven Development** - 常に最初にテストを書く
- **Systematic over ad-hoc** - 当てずっぽうよりプロセス
- **Complexity reduction** - 単純さを最優先の目標にする
- **Evidence over claims** - 成功を宣言する前に検証する

[元のリリース告知](https://blog.fsck.com/2025/10/09/superpowers/) を読んでください。

## コントリビュート

Superpowers の一般的なコントリビューション手順は以下のとおりです。新しいスキルのコントリビューションは通常受け付けておらず、スキルの更新は、私たちがサポートしているすべてのコーディングエージェントで動作する必要があることに留意してください。

1. リポジトリを fork する
2. `dev` ブランチに切り替える
3. 作業用ブランチを作成する
4. 新規および変更したスキルの作成とテストには `writing-skills` スキルに従う
5. PR を送る。このとき pull request template を必ず埋める

完全なガイドは `skills/writing-skills/SKILL.md` を参照してください。

## 更新

Superpowers の更新方法はコーディングエージェントへの依存がややありますが、多くの場合は自動です。

## ライセンス

MIT License - 詳細は LICENSE ファイルを参照してください

## コミュニティ

Superpowers は [Jesse Vincent](https://blog.fsck.com) と [Prime Radiant](https://primeradiant.com) の仲間たちによって作られています。

- **Discord**: コミュニティサポート、質問、そして Superpowers で何を作っているかの共有のために [参加してください](https://discord.gg/35wsABTejz)
- **Issues**: https://github.com/obra/superpowers/issues
- **Release announcements**: 新しいバージョンの通知を受け取るには [登録してください](https://primeradiant.com/superpowers/)
