# prompt-smith

Claude Code プラグイン [prompt-smith](https://github.com/nyoki/my-claude-plugins) の開発・保守用リポジトリ。

プラグイン本体は `my-claude-plugins/plugins/prompt-smith/` にあり、このリポジトリはその更新に使う **参考資料の管理** と **パターン更新コマンド** を提供する。

## このリポジトリでやること

- Claude Code 公式ドキュメントの仕様情報を参考資料として管理する（権威ソース）
- Anthropic 公式リポジトリのプロンプトパターンを分析し、プラグインのパターン定義を最新に保つ

## ディレクトリ構成

```
prompt-smith/
├── .claude/
│   └── commands/
│       └── update-patterns.md         # パターン更新コマンド
├── docs/
│   ├── official-reference.md          # 公式ドキュメント仕様リファレンス（権威ソース）
│   └── prompt-creation-guide.md       # パターン分析結果（公式リポから抽出）
├── report.md                          # 類似公式プラグインとの比較レポート
└── .refs/                             # 分析対象リポジトリ（gitignored、自動取得）
```

### docs/ の使い分け

| ファイル | 内容 | 観点 |
|---------|------|------|
| `official-reference.md` | Claude Code 公式ドキュメントの仕様情報 | 「何が正しいか」（仕様・権威） |
| `prompt-creation-guide.md` | 公式リポのプロンプトを統計分析した結果 | 「何が一般的か」（パターン・補完） |

## 開発ワークフロー

### 1. パターン分析・更新

```bash
# Claude Code で実行
/update-patterns

# 分析のみ（ファイル変更なし）
/update-patterns --analyze-only
```

このコマンドは以下を自動で行う：
1. 公式ドキュメントを WebFetch で取得・分析
2. 公式リポジトリを `.refs/` に clone/pull し、パターンを分析
3. 両ソースを突合し、`my-claude-plugins/plugins/prompt-smith/` 内の定義を更新

### 2. 動作確認

```bash
# バージョン確認
/prompt-smith:version

# レビュー機能のテスト
/prompt-smith:review-prompt @path/to/prompt.md
```

## 参考ソース

本プロジェクトは以下のソースを参考にしている（優先度順）：

1. [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code/) - Skills / Agents / Plugins の仕様（権威ソース）
2. [anthropics/claude-code](https://github.com/anthropics/claude-code) - Claude Code 本体（(C) Anthropic PBC, All Rights Reserved）
3. [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) - Anthropic 公式プラグインディレクトリ

リポジトリのコードは本リポジトリに含まれていない。分析時に `.refs/` に自動取得され、gitignore で除外されている。

## ライセンス

[MIT](./LICENSE)
