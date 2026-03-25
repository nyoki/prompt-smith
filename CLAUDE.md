# CLAUDE.md

## このリポジトリについて

prompt-smith は **Claude Code プラグイン `prompt-smith` の開発・保守用作業リポジトリ** である。
プラグイン本体は [my-claude-plugins](https://github.com/nyoki/my-claude-plugins) リポジトリの `plugins/prompt-smith/` に配置されている。

**このリポジトリの役割**:
- Claude Code 公式ドキュメントの仕様情報を参考資料として管理する
- 公式リポジトリのプロンプトパターンを分析し、プラグインのパターン定義を最新に保つ
- `/update-patterns` コマンドで上記をプラグイン側に反映する

**このリポジトリでは行わないこと**:
- プラグインのコード（skills/agents/commands）を直接編集する（それは my-claude-plugins 側の作業）

## ディレクトリ構成

```
prompt-smith/
├── CLAUDE.md                          # このファイル
├── README.md
├── .claude/
│   └── commands/
│       └── update-patterns.md         # パターン更新コマンド（メインワークフロー）
├── docs/
│   ├── official-reference.md          # 公式ドキュメント仕様リファレンス（権威ソース）
│   └── prompt-creation-guide.md       # パターン分析結果（公式リポから抽出したパターン）
├── report.md                          # 類似公式プラグインとの比較レポート
└── .refs/                             # 分析対象リポジトリ（gitignored、自動取得）
    ├── claude-code/
    └── claude-plugins-official/
```

## docs/ の役割

| ファイル | 内容 | 更新方法 |
|---------|------|---------|
| `official-reference.md` | Claude Code 公式ドキュメントから抽出した仕様情報（何が正しいか） | `/update-patterns` で自動更新 |
| `prompt-creation-guide.md` | 公式リポジトリのプロンプトを統計的に分析した結果（何が多いか、何が一般的か） | `/update-patterns` で自動更新 |

プラグインを更新する際は、**両方を参照**して判断する。矛盾がある場合は公式リファレンス（仕様）が優先。パターン分析は「現実」を補完する。

## 関連リソース

| リソース | 種別 | 関係 |
|---------|------|------|
| `my-claude-plugins/plugins/prompt-smith/` | プラグイン | 更新先 |
| Claude Code 公式ドキュメント | 仕様（権威） | WebFetch で取得 |
| `anthropics/claude-code` | リポジトリ | `.refs/` に clone |
| `anthropics/claude-plugins-official` | リポジトリ | `.refs/` に clone |

## ワークフロー

1. `/update-patterns` を実行
2. 公式ドキュメントを WebFetch で取得・分析
3. 公式リポジトリを `.refs/` に clone/pull し、パターンを分析
4. `docs/official-reference.md` と `docs/prompt-creation-guide.md` を参照しつつ、my-claude-plugins 側のパターン定義を更新
