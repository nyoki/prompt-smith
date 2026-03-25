# Claude Code 公式ドキュメント リファレンス

このドキュメントは Claude Code 公式ドキュメントから抽出した Skill / Agent / Command / Plugin の仕様情報をまとめたものである。
`/update-patterns` でプラグインを更新する際、パターン分析結果（`prompt-creation-guide.md`）と合わせて正確な判断材料として使用する。

**出典**: https://code.claude.com/docs/en/ （旧 docs.anthropic.com から移行済み）
**最終更新**: 2026/03/26

---

## 1. Skill

### 概要

- Skill は Claude の能力を拡張する仕組み。`SKILL.md` ファイルを作成すると Claude のツールキットに追加される
- `/skill-name` で直接呼び出すか、Claude が文脈から自動的に使用する
- **Commands は Skills に統合済み**: `.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` は同等。既存の `.claude/commands/` ファイルはそのまま機能する（後方互換）
- [Agent Skills](https://agentskills.io) オープンスタンダードに準拠

### ファイル構造

```
my-skill/
├── SKILL.md           # メイン指示（必須）
├── template.md        # Claude が埋めるテンプレート（任意）
├── examples/
│   └── sample.md      # 期待されるフォーマットのサンプル（任意）
└── scripts/
    └── validate.sh    # Claude が実行できるスクリプト（任意）
```

### 配置場所と優先順位

| 優先度 | 場所 | パス | 適用範囲 |
|--------|------|------|----------|
| 1（最高） | Enterprise | managed settings 参照 | 組織全ユーザー |
| 2 | Personal | `~/.claude/skills/<skill-name>/SKILL.md` | 全プロジェクト |
| 3 | Project | `.claude/skills/<skill-name>/SKILL.md` | そのプロジェクトのみ |
| - | Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | プラグイン有効な場所（名前空間で分離） |

Plugin skills は `plugin-name:skill-name` 名前空間を使用するため競合しない。

### YAML Frontmatter フィールド

| フィールド | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| `name` | No | ディレクトリ名 | スキルの表示名。小文字・数字・ハイフンのみ（最大64文字） |
| `description` | 推奨 | Markdown の最初の段落 | スキルの説明と使用タイミング。Claude が自動使用の判断に利用 |
| `argument-hint` | No | - | オートコンプリート時のヒント。例: `[issue-number]`, `[filename] [format]` |
| `disable-model-invocation` | No | `false` | `true` で Claude の自動ロードを防ぐ |
| `user-invocable` | No | `true` | `false` で `/` メニューから非表示にする |
| `allowed-tools` | No | - | スキル有効時に Claude が許可なく使えるツール |
| `model` | No | セッションから継承 | スキル有効時に使用するモデル |
| `effort` | No | セッションから継承 | 努力レベル: `low`, `medium`, `high`, `max`（Opus 4.6 のみ） |
| `context` | No | - | `fork` でフォークされたサブエージェントコンテキストで実行 |
| `agent` | No | - | `context: fork` 設定時に使用するサブエージェントの種類 |
| `hooks` | No | - | このスキルのライフサイクルにスコープされたフック |

### 文字列置換

| 変数 | 説明 |
|------|------|
| `$ARGUMENTS` | スキル呼び出し時に渡された全引数。コンテンツに `$ARGUMENTS` がなければ末尾に `ARGUMENTS: <value>` として追加 |
| `$ARGUMENTS[N]` | 0ベースインデックスで特定の引数にアクセス |
| `$N` | `$ARGUMENTS[N]` の短縮形。例: `$0`, `$1` |
| `${CLAUDE_SESSION_ID}` | 現在のセッション ID |
| `${CLAUDE_SKILL_DIR}` | スキルの `SKILL.md` が含まれるディレクトリ |

### 呼び出し制御の動作

| Frontmatter 設定 | ユーザーが呼び出し可能 | Claude が呼び出し可能 | コンテキストへの読み込み |
|------------------|----------------------|---------------------|------------------------|
| （デフォルト） | Yes | Yes | 説明は常にコンテキストに。フル内容は呼び出し時にロード |
| `disable-model-invocation: true` | Yes | No | 説明はコンテキストに含まれない。ユーザーが呼び出したときのみロード |
| `user-invocable: false` | No | Yes | 説明は常にコンテキストに。フル内容は呼び出し時にロード |

### 動的コンテキスト注入

`` !`<command>` `` 構文でシェルコマンドをスキルコンテンツ送信前に実行:

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`
```

### ベストプラクティス（公式）

- `SKILL.md` は **500行以内** に保つ。詳細なリファレンス資料は別ファイルへ
- リファレンスファイルは `SKILL.md` から参照して、Claude がいつロードするかを知らせる
- 副作用のあるワークフロー（`/commit`, `/deploy`）は `disable-model-invocation: true` を使用
- サブエージェントで `context: fork` を使う場合、スキルには明示的な指示が必要（ガイドラインのみだと意味がない）
- Extended thinking を有効にするには、スキルコンテンツに "ultrathink" という単語を含める

### コンテキストバジェット

- スキル説明はコンテキストウィンドウの 2% に動的スケール（フォールバック: 16,000 文字）
- 環境変数 `SLASH_COMMAND_TOOL_CHAR_BUDGET` でオーバーライド可能

### 組み込みスキル

| スキル | 目的 |
|--------|------|
| `/batch <instruction>` | コードベース全体の大規模変更を並列で調整 |
| `/claude-api` | Claude API リファレンス資料をロード |
| `/debug [description]` | 現在のセッションのデバッグ |
| `/loop [interval] <prompt>` | プロンプトをインターバルで繰り返し実行 |
| `/simplify [focus]` | 最近変更されたファイルのコード品質改善 |

---

## 2. Agent（サブエージェント）

### 概要

- サブエージェントは特定タスクを処理する専用 AI アシスタント
- 独自のコンテキストウィンドウ、カスタムシステムプロンプト、特定ツールアクセス、独立した権限で動作
- Claude はタスクがサブエージェントの説明に一致する場合、そのサブエージェントに委任

### 配置場所と優先順位

| 優先度 | 場所 | パス |
|--------|------|------|
| 1（最高） | CLI フラグ | `--agents` |
| 2 | Project | `.claude/agents/` |
| 3 | Personal | `~/.claude/agents/` |
| 4（最低） | Plugin | `<plugin>/agents/` |

### YAML Frontmatter フィールド

| フィールド | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| `name` | Yes | - | 小文字とハイフンの一意識別子 |
| `description` | Yes | - | Claude がいつこのサブエージェントに委任するかの説明 |
| `tools` | No | 全ツールを継承 | 使用できるツール（Allowlist） |
| `disallowedTools` | No | - | 拒否するツール（Denylist） |
| `model` | No | `inherit` | 使用モデル: `sonnet`, `opus`, `haiku`, フルモデル ID, `inherit` |
| `permissionMode` | No | - | 権限モード: `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | - | 最大エージェントターン数 |
| `skills` | No | - | 起動時にコンテキストに注入するスキル（フルコンテンツが注入される） |
| `mcpServers` | No | - | 利用できる MCP サーバー |
| `hooks` | No | - | ライフサイクルフック |
| `memory` | No | - | 永続メモリスコープ: `user`, `project`, `local` |
| `background` | No | `false` | `true` で常にバックグラウンドタスクとして実行 |
| `effort` | No | セッションから継承 | 努力レベル: `low`, `medium`, `high`, `max` |
| `isolation` | No | - | `worktree` で一時的な git worktree で実行 |

### ツール制御

```yaml
# Allowlist（tools フィールド）
tools: Read, Grep, Glob, Bash

# Denylist（disallowedTools フィールド）
disallowedTools: Write, Edit

# 子エージェントのスポーン制限
tools: Agent(worker, researcher), Read, Bash
```

両方設定時: `disallowedTools` が先に適用され、その後 `tools` が解決される。

### 永続メモリスコープ

| スコープ | 保存場所 | 使用場面 |
|---------|----------|---------|
| `user` | `~/.claude/agent-memory/<name>/` | 全プロジェクト共通の学習 |
| `project` | `.claude/agent-memory/<name>/` | プロジェクト固有（バージョン管理共有可） |
| `local` | `.claude/agent-memory-local/<name>/` | プロジェクト固有（バージョン管理外） |

メモリ有効時の動作:
- サブエージェントのシステムプロンプトにメモリディレクトリの読み書き指示が含まれる
- `MEMORY.md` の最初の 200 行がシステムプロンプトに含まれる
- Read, Write, Edit ツールが自動的に有効化

### CLI からの定義

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer...",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

`prompt` がマークダウン本文に相当する。

### プラグインエージェントのセキュリティ制限

プラグイン提供のサブエージェントは、セキュリティ上の理由から以下をサポート**しない**:
- `hooks`
- `mcpServers`
- `permissionMode`

### フォアグラウンド vs バックグラウンド

- **フォアグラウンド**: 完了するまでメイン会話をブロック。権限プロンプトはユーザーに渡される
- **バックグラウンド**: 作業を続けながら並列実行。起動前に必要なツール権限を確認

### コンテキスト管理

- 各起動で新しいインスタンスとフレッシュコンテキストが作成される
- 再開（resume）で以前の完全な会話履歴を保持
- トランスクリプトは `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl` に保存
- デフォルトで 30 日後に自動クリーンアップ（`cleanupPeriodDays` 設定）

### ベストプラクティス（公式）

- 各サブエージェントは **1つの特定タスクに特化** させる
- Claude が委任タイミングを判断するため、**明確な description** を書く
- セキュリティとフォーカスのため **必要な権限のみ** 付与
- プロジェクトサブエージェントは **バージョン管理にコミット** してチームで共有

### 組み込みサブエージェント

| エージェント | モデル | ツール | 目的 |
|------------|--------|--------|------|
| Explore | Haiku | 読み取り専用 | コードベース探索・検索 |
| Plan | 親から継承 | 読み取り専用 | プランモードでのコードベース調査 |
| General-purpose | 親から継承 | 全ツール | 複雑なマルチステップタスク |
| Bash | 継承 | - | 独立コンテキストでターミナルコマンド実行 |
| Claude Code Guide | Haiku | - | Claude Code 機能についての質問 |

---

## 3. Command（スラッシュコマンド）

### Skills への統合

- Custom commands は Skills に統合された（後方互換性は維持）
- `.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` は同等
- 新規作成時は Skills として作成することが推奨される

### 後方互換のフィールド

| 新しいフィールド | 旧フィールド | 説明 |
|----------------|-------------|------|
| `user-invocable: false` | `hide-from-slash-command-tool: "true"` | `/` メニューから非表示 |

---

## 4. Plugin

### 概要

- プラグインはスキル、エージェント、フック、MCP サーバーを含む自己完結型の拡張ディレクトリ
- `.claude-plugin/plugin.json` を含むディレクトリがプラグインとして認識される

### Standalone vs Plugin

| 方式 | スキル名 | 最適用途 |
|------|---------|---------|
| Standalone（`.claude/` ディレクトリ） | `/hello` | 個人ワークフロー、単一プロジェクト、実験 |
| Plugin（`.claude-plugin/plugin.json` 含む） | `/plugin-name:hello` | チーム共有、コミュニティ配布、複数プロジェクト再利用 |

### 標準ディレクトリ構造

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json           # プラグインマニフェスト
├── commands/                 # コマンド（デフォルト場所）
├── agents/                   # エージェント（デフォルト場所）
├── skills/                   # スキル（デフォルト場所）
├── hooks/
│   └── hooks.json            # フック設定
├── settings.json             # プラグイン有効時のデフォルト設定（現在 `agent` キーのみ対応）
├── .mcp.json                 # MCP サーバー定義
├── .lsp.json                 # LSP サーバー設定（コードインテリジェンス）
├── scripts/                  # フック・ユーティリティスクリプト
├── LICENSE
└── CHANGELOG.md
```

**重要**: `.claude-plugin/` には `plugin.json` のみ配置。`commands/`, `agents/`, `skills/`, `hooks/` は全てプラグインルートに置く。

### LSP サーバー

プラグインは `.lsp.json` で Language Server Protocol サーバーを提供し、Claude にリアルタイムのコードインテリジェンス（診断、定義へ移動、参照検索）を提供できる。

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

必須フィールド: `command`, `extensionToLanguage`
オプション: `args`, `transport`（`stdio`/`socket`）, `env`, `initializationOptions`, `settings`, `workspaceFolder`, `startupTimeout`, `shutdownTimeout`, `restartOnCrash`, `maxRestarts`

### settings.json（プラグインデフォルト設定）

プラグインルートに `settings.json` を配置すると、プラグイン有効時にデフォルト設定が適用される。現在は `agent` キーのみサポート。

```json
{
  "agent": "security-reviewer"
}
```

### plugin.json マニフェスト

#### 必須フィールド

| フィールド | 型 | 説明 |
|-----------|------|------|
| `name` | string | 一意識別子（kebab-case）。スキル名前空間として使用 |

#### メタデータフィールド

| フィールド | 型 | 説明 |
|-----------|------|------|
| `version` | string | セマンティックバージョン |
| `description` | string | プラグインの目的 |
| `author` | object | 作者情報（`name`, `email`, `url`） |
| `homepage` | string | ドキュメント URL |
| `repository` | string | ソースコード URL |
| `license` | string | ライセンス識別子 |
| `keywords` | array | 発見用タグ |

#### コンポーネントパスフィールド

| フィールド | 型 | 説明 |
|-----------|------|------|
| `commands` | string\|array | 追加コマンドファイル/ディレクトリ |
| `agents` | string\|array | 追加エージェントファイル |
| `skills` | string\|array | 追加スキルディレクトリ |
| `hooks` | string\|array\|object | フック設定パスまたはインライン設定 |
| `mcpServers` | string\|array\|object | MCP 設定パスまたはインライン設定 |
| `outputStyles` | string\|array | 追加出力スタイルファイル/ディレクトリ |
| `lspServers` | string\|array\|object | LSP サーバー設定（コードインテリジェンス: 定義へ移動、参照検索等） |

カスタムパスはデフォルトディレクトリに**加えて追加**される（置き換えではない）。全パスは `./` で始まりプラグインルートからの相対パス。

### 環境変数

| 変数 | 説明 |
|------|------|
| `${CLAUDE_PLUGIN_ROOT}` | プラグインのインストールディレクトリへの絶対パス |
| `${CLAUDE_PLUGIN_DATA}` | 更新をまたいで永続するデータ用ディレクトリ（`~/.claude/plugins/data/{id}/`） |

### フックイベント一覧

| イベント | 発火タイミング |
|----------|--------------|
| `SessionStart` | セッション開始/再開時 |
| `UserPromptSubmit` | プロンプト送信時（Claude が処理する前） |
| `PreToolUse` | ツール呼び出し実行前（ブロック可能） |
| `PermissionRequest` | 権限ダイアログ表示時 |
| `PostToolUse` | ツール呼び出し成功後 |
| `PostToolUseFailure` | ツール呼び出し失敗後 |
| `Notification` | 通知送信時 |
| `SubagentStart` | サブエージェント起動時 |
| `SubagentStop` | サブエージェント終了時 |
| `Stop` | Claude の応答終了時 |
| `StopFailure` | API エラーでターン終了時 |
| `TeammateIdle` | エージェントチームメンバーがアイドル状態時 |
| `TaskCompleted` | タスクが完了としてマーク時 |
| `InstructionsLoaded` | CLAUDE.md または `.claude/rules/*.md` がロード時 |
| `ConfigChange` | 設定ファイルがセッション中に変更時 |
| `WorktreeCreate` | worktree 作成時 |
| `WorktreeRemove` | worktree 削除時 |
| `PreCompact` | コンテキストコンパクション前 |
| `PostCompact` | コンテキストコンパクション完了後 |
| `Elicitation` | MCP サーバーがユーザー入力を要求時 |
| `ElicitationResult` | ユーザーが MCP elicitation に応答後 |
| `SessionEnd` | セッション終了時 |

### フックタイプ

| タイプ | 説明 |
|--------|------|
| `command` | シェルコマンド/スクリプトを実行 |
| `http` | イベント JSON を URL に POST |
| `prompt` | LLM でプロンプトを評価（`$ARGUMENTS` 使用） |
| `agent` | ツールを持つエージェント型バリファイアを実行（複雑な検証タスク向け） |

### フック設定例

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format-code.sh"
          }
        ]
      }
    ]
  }
}
```

### インストールスコープ

| スコープ | 設定ファイル | ユースケース |
|---------|------------|------------|
| `user` | `~/.claude/settings.json` | 全プロジェクトで使える個人プラグイン（デフォルト） |
| `project` | `.claude/settings.json` | バージョン管理で共有するチームプラグイン |
| `local` | `.claude/settings.local.json` | gitignore されるプロジェクト固有プラグイン |

### バージョン管理

- セマンティックバージョニング（`MAJOR.MINOR.PATCH`）に従う
- バージョンを上げないとキャッシュにより既存ユーザーに変更が届かない
- `1.0.0` から開始
- 変更は `CHANGELOG.md` に記録

### テストとデバッグ

```bash
# ローカルテスト
claude --plugin-dir ./my-plugin

# 複数プラグイン
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two

# 変更反映（再起動不要）
/reload-plugins

# バリデーション
claude plugin validate
```

### よくある問題

| 問題 | 原因 | 解決策 |
|------|------|--------|
| プラグインが読み込まれない | 無効な `plugin.json` | `claude plugin validate` を実行 |
| コマンドが表示されない | ディレクトリ構造が間違い | `commands/` がルートにあることを確認 |
| フックが発火しない | スクリプトが実行可能でない | `chmod +x script.sh` |
| MCP サーバーが失敗 | `${CLAUDE_PLUGIN_ROOT}` なし | 全プラグインパスにこの変数を使用 |
| パスエラー | 絶対パスが使用されている | 全パスを `./` からの相対パスに |

---

## 5. Skill と Agent の関係

| アプローチ | システムプロンプト | タスク | ロードされるもの |
|-----------|-----------------|--------|----------------|
| `context: fork` を持つ Skill | エージェントタイプから（Explore, Plan 等） | SKILL.md の内容 | CLAUDE.md |
| `skills` フィールドを持つ Subagent | Subagent のマークダウン本文 | Claude の委任メッセージ | プリロードスキル + CLAUDE.md |
