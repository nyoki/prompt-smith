# Claude Code プロンプト作成ガイドライン

このドキュメントは、claude-code公式リポジトリのプロンプト分析に基づいて作成されました。

## 目的

1. **プロンプトのレビュー**: 自作プロンプトの品質チェック
2. **プロンプト生成**: AIがプロンプトを作成する際のインプット

---

## カテゴリ概要

| カテゴリ | 用途 | トリガー方式 |
|----------|------|--------------|
| **Skill** | ドメイン知識・ガイダンス提供 | 自動（description内のフレーズにマッチ） |
| **Agent** | 特定タスクの自律実行 | Task tool経由で起動 |
| **Command** | ユーザー起動のワークフロー | `/command`で明示的に実行 |

---

## 1. Skill 作成ガイド

### 概要
Skillは「知識の提供」が目的。ユーザーが特定のトピックについて質問したとき、関連するガイダンスを提供する。

### YAML Frontmatter 構造

```yaml
---
name: skill-name
description: This skill should be used when the user asks to "phrase 1", "phrase 2", "phrase 3" or needs guidance on [topic].
version: 0.1.0
---
```

#### フィールド説明

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `name` | ✅ | スキル識別子（ハイフン区切り） |
| `description` | ✅ | トリガー条件（三人称形式で記述） |
| `version` | ✅ | セマンティックバージョン |

#### description の書き方

```yaml
# ✅ 正しい形式
description: This skill should be used when the user asks to "create a hook", "add a PreToolUse hook", or mentions hook events (PreToolUse, PostToolUse).

# ❌ 誤った形式
description: Use this skill when you want to create hooks.
```

**ポイント**:
- 必ず "This skill should be used when..." で開始
- 具体的なユーザーフレーズを引用符で列挙
- 技術用語・イベント名も明示

### 本文構造

```markdown
# [Skill Name]

## Overview
[スキルの目的と概要 - 1-2段落]

## Key Concepts
[核となる概念の説明]

## [Main Content Sections]
[スキル固有の詳細]

## Best Practices

**DO:**
- ✅ [推奨事項1]
- ✅ [推奨事項2]

**DON'T:**
- ❌ [避けるべき事項1]
- ❌ [避けるべき事項2]

## Quick Reference
[テーブル形式のクイックリファレンス]

## Troubleshooting
[よくある問題と解決策]

## Additional Resources

### Reference Files
- **`references/patterns.md`** - 詳細なパターン
- **`references/advanced.md`** - 高度なテクニック

### Example Files
- **`examples/example.sh`** - 動作する例
```

### 文体ルール

| 場所 | 形式 | 例 |
|------|------|-----|
| Frontmatter description | 三人称 | "This skill should be used when..." |
| 本文 | 命令形 | "Configure the server.", "Define the event type." |

### Progressive Disclosure

```
skill-name/
├── SKILL.md          # 1,500-2,000語（最大5,000語）
├── references/       # 詳細ドキュメント（必要時にロード）
│   ├── patterns.md
│   └── advanced.md
├── examples/         # 動作するコード例
└── scripts/          # ユーティリティスクリプト
```

### チェックリスト

- [ ] Frontmatterに name, description, version がある
- [ ] descriptionが "This skill should be used when..." で始まる
- [ ] 具体的なトリガーフレーズが列挙されている
- [ ] 本文は命令形で記述されている
- [ ] SKILL.mdは5,000語以下
- [ ] 詳細な情報はreferences/に分離されている
- [ ] 動作する例がexamples/にある

---

## 2. Agent 作成ガイド

### 概要
Agentは「タスクの実行」が目的。特定の条件でTask toolから起動され、自律的にタスクを完了する。

### YAML Frontmatter 構造

```yaml
---
name: agent-name
description: |
  Use this agent when [triggering conditions].
  Examples:
  <example>
  Context: [状況説明]
  user: "[ユーザーメッセージ]"
  assistant: "[起動前の応答]"
  <commentary>
  [なぜこのエージェントを使うべきか]
  </commentary>
  assistant: "I'll use the agent-name agent to [アクション]."
  </example>
model: sonnet
color: cyan
tools: ["Read", "Grep", "Glob"]
---
```

#### フィールド説明

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `name` | ✅ | エージェント識別子（小文字、ハイフン区切り、3-50文字） |
| `description` | ✅ | トリガー条件 + 2-4個の`<example>`ブロック |
| `model` | ❌ | `sonnet`, `opus`, `haiku`, `inherit`（デフォルト） |
| `color` | ❌ | 視覚識別子（blue/cyan/yellow/green/red/magenta） |
| `tools` | ❌ | 使用可能ツールの配列（省略時は全ツール） |

#### モデル選択ガイド

| モデル | 用途 |
|--------|------|
| `sonnet` | 作成・設計・実装（code-architect, agent-creator） |
| `opus` | 精密なレビュー・バグ検出（code-reviewer, silent-failure-hunter） |
| `haiku` | 探索・簡単なチェック |
| `inherit` | コンテキストに応じて判断 |

#### ツール設定パターン

```yaml
# 分析系（読み取り専用）
tools: ["Read", "Grep", "Glob"]

# 作成系（書き込み可能）
tools: ["Write", "Read"]

# アーキテクチャ系（フルアクセス）
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch
```

### 本文構造

```markdown
You are an expert [専門分野] specializing in [具体的な専門性].

## Core Responsibilities

1. [責任1]
2. [責任2]
3. [責任3]

## Process

### Step 1: [ステップ名]
[詳細な手順]

### Step 2: [ステップ名]
[詳細な手順]

## Quality Standards

- [基準1]
- [基準2]

## Output Format

```markdown
## [Section Title]

### Summary
[概要]

### Findings
[発見事項]

### Recommendations
1. [優先度高]
2. [優先度中]
3. [優先度低]
```

## Edge Cases

- [エッジケース1の処理方法]
- [エッジケース2の処理方法]
```

### description の例（2-4個のexampleが必須）

```yaml
description: |
  Use this agent when the user wants to analyze code for feature implementation or needs to trace code flows across the codebase.

  Examples:

  <example>
  Context: User is exploring a new codebase
  user: "How is authentication implemented?"
  assistant: "I'll use the code-explorer agent to trace the authentication flow."
  <commentary>
  This requires tracing code across multiple files, which is the agent's specialty.
  </commentary>
  </example>

  <example>
  Context: User needs to understand a feature
  user: "Where is the payment processing handled?"
  assistant: "Let me launch the code-explorer agent to find all payment-related code."
  <commentary>
  Payment processing likely spans multiple modules, requiring comprehensive exploration.
  </commentary>
  </example>
```

### チェックリスト

- [ ] nameが小文字・ハイフン区切り・3-50文字
- [ ] descriptionに2-4個の`<example>`ブロックがある
- [ ] 各exampleにContext, user, assistant, commentaryがある
- [ ] 本文が専門家ペルソナで始まる
- [ ] 責任・プロセス・品質基準が明確
- [ ] 出力フォーマットが具体的に定義されている
- [ ] エッジケースが考慮されている
- [ ] モデル選択が目的に適している

---

## 3. Command 作成ガイド

### 概要
Commandは「ワークフローの実行」が目的。`/command`でユーザーが明示的に起動する。

### YAML Frontmatter 構造

```yaml
---
description: コマンドの簡潔な説明
argument-hint: "[引数のヒント]"
allowed-tools: ["Read", "Write", "Bash(git:*)"]
---
```

#### フィールド説明

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `description` | ✅ | ヘルプに表示される説明文 |
| `argument-hint` | ❌ | 引数の形式ヒント |
| `allowed-tools` | ❌ | 使用可能ツールのホワイトリスト |
| `hide-from-slash-command-tool` | ❌ | `"true"`で非表示 |

#### allowed-tools の書き方

```yaml
# 特定のbashコマンドのみ許可
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)

# ツール配列
allowed-tools: ["Read", "Write", "Grep", "Glob", "TodoWrite"]

# プラグインルート相対パス
allowed-tools: ["Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup.sh)"]
```

### 本文構造パターン

#### Pattern A: フェーズ構造（複雑なワークフロー）

```markdown
## Phase 1: Discovery
**Goal**: [フェーズの目的]
**Actions**:
1. [アクション1]
2. [アクション2]
**Output**: [期待される成果物]

**Wait for user confirmation before proceeding.**

## Phase 2: Planning
...
```

#### Pattern B: 手順的リスト（中程度の複雑さ）

```markdown
Follow these steps precisely:

1. [ステップ1の詳細]
2. [ステップ2の詳細]
3. [ステップ3の詳細]
```

#### Pattern C: シンプルな命令（単純なタスク）

```markdown
## Context

- Current git status: !`git status`
- Current branch: !`git branch --show-current`

## Your Task

Based on the above:
1. [タスク1]
2. [タスク2]
```

### $ARGUMENTS の扱い方

```markdown
**Initial request:** $ARGUMENTS

**If $ARGUMENTS is provided:**
- User has given specific instructions: `$ARGUMENTS`
- Proceed with the specified task

**If $ARGUMENTS is empty:**
- Launch discovery phase
- Ask user for clarification
```

### Agent/Skill 統合

```markdown
## Phase 2: Exploration

**Load required skills:**
- Use Skill tool to load `plugin-structure` skill

**Launch parallel agents:**
1. Agent 1 (haiku): Explore existing patterns
2. Agent 2 (sonnet): Analyze architecture

**After agents return:**
- Read all identified files
- Synthesize findings
```

### エラーハンドリング

```markdown
## Pre-conditions

1. Launch a haiku agent to check if:
   - The pull request is closed
   - The pull request is a draft
   If any condition is true, stop and explain why.

## Validation

After implementation:
- Run `npx tsc --noEmit` to check for type errors
- Fix ALL errors before proceeding
- **DO NOT consider complete until validation passes**
```

### 動的コンテキスト

```markdown
## Context

- Current git status: !`git status`
- Current git diff: !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10`
```

### チェックリスト

- [ ] descriptionが簡潔で明確
- [ ] argument-hintが引数の使い方を示している
- [ ] allowed-toolsで必要なツールのみ許可
- [ ] ワークフローが論理的な順序で構成されている
- [ ] $ARGUMENTSの扱いが明確
- [ ] ユーザー確認ポイントが適切に配置されている
- [ ] エラーハンドリングが考慮されている
- [ ] 検証ステップが含まれている

---

## 共通ベストプラクティス

### 1. 明確さ

```markdown
# ✅ 良い例
Configure the MCP server with authentication using OAuth2.

# ❌ 悪い例
You should probably set up the server somehow.
```

### 2. 具体性

```markdown
# ✅ 良い例
Run `npx tsc --noEmit` to verify types pass.

# ❌ 悪い例
Make sure the code works.
```

### 3. 例の活用

````markdown
**Example:**
```typescript
const config: Config = {
  timeout: 5000,
  retries: 3
};
```

**Why this works:** Explicit typing prevents runtime errors.
````

### 4. 比較による説明

```markdown
❌ **Bad:**
```javascript
const data = fetchData();
```
**Why bad:** No error handling.

✅ **Good:**
```javascript
try {
  const data = await fetchData();
} catch (e) {
  handleError(e);
}
```
**Why good:** Handles failures gracefully.
```

### 5. チェックリスト形式

```markdown
**Before Proceeding:**
- [ ] All tests pass
- [ ] Types are verified
- [ ] No console.log statements remain
```

---

## プロンプトレビューチェックリスト

### 構造
- [ ] YAML Frontmatterが正しい形式
- [ ] 必須フィールドがすべて存在
- [ ] 本文が論理的に構成されている

### 内容
- [ ] 目的が明確に述べられている
- [ ] 具体的な例が含まれている
- [ ] エッジケースが考慮されている
- [ ] 検証・完了条件が明確

### 文体
- [ ] 一貫した形式（命令形/三人称）
- [ ] 簡潔で明確な表現
- [ ] 技術的に正確

### Progressive Disclosure
- [ ] 本文は適切なサイズ（Skill: <5,000語）
- [ ] 詳細は参照ファイルに分離
- [ ] 参照ファイルが明示的にリストされている

---

## AIへのプロンプト生成指示

以下のテンプレートを使用して、AIにプロンプト生成を依頼できます：

```
あなたはClaude Codeの[Skill/Agent/Command]を作成する専門家です。

以下の要件に基づいて[Skill/Agent/Command]を作成してください：

## 要件
- 目的: [具体的な目的]
- トリガー条件: [いつ起動されるべきか]
- 必要なツール: [使用するツール]
- 期待する出力: [出力形式]

## 制約
- [制約1]
- [制約2]

## 作成ルール
1. YAML Frontmatterは本ガイドラインに従う
2. 本文は[命令形/三人称]で記述
3. 具体的な例を2-4個含める
4. エッジケースを考慮する
5. 検証ステップを含める

上記のprompt-creation-guide.mdのフォーマットとベストプラクティスに従って作成してください。
```
