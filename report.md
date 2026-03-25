# prompt-smith 類似公式プラグイン調査レポート

作成日: 2026/03/16

## 1. prompt-smith と類似する公式プラグイン

prompt-smith は「Claude Code の Skill / Agent / Command プロンプトの作成・レビュー・生成」を目的としたプラグインである。
以下の公式プラグインが機能的に重複・類似している。

### 1-1. skill-creator（Anthropic 公式）

- **リポジトリ**: [anthropics/claude-plugins-official/plugins/skill-creator](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator)
- **インストール**: `/plugin install skill-creator@claude-plugin-directory`
- **概要**: スキルの作成・テスト・改善・ベンチマークを行う包括的なツールキット
- **4つの動作モード**:
  - **Create** — インタビュー → ドラフト作成 → テストケース生成
  - **Eval** — サブエージェントで並列テスト実行、ベースラインとの比較
  - **Improve** — ユーザーフィードバックに基づくスキル改善ループ
  - **Benchmark** — 複数回実行での定量的パフォーマンス測定（pass_rate, tokens, duration）
- **付属ツール**:
  - `eval-viewer/generate_review.py` — ブラウザベースの結果レビューUI
  - `scripts/run_loop.py` — description 最適化の自動ループ
  - `scripts/aggregate_benchmark.py` — ベンチマーク集計
  - `scripts/package_skill.py` — .skill ファイルへのパッケージング
  - `agents/grader.md`, `agents/comparator.md`, `agents/analyzer.md` — 評価用サブエージェント
- **対象**: **Skill のみ**（Agent / Command は対象外）

### 1-2. claude-md-management（Anthropic 公式）

- **リポジトリ**: [anthropics/claude-plugins-official/plugins/claude-md-management](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management)
- **概要**: CLAUDE.md ファイルの品質監査と改善
- **構成**:
  - **claude-md-improver（Skill）** — CLAUDE.md を6軸（Commands/workflows, Architecture clarity, Non-obvious patterns, Conciseness, Currency, Actionability）で100点満点評価し、改善提案
  - **/revise-claude-md（Command）** — セッション中の学びを CLAUDE.md に反映
- **対象**: CLAUDE.md の管理に特化（Skill/Agent/Command プロンプトの作成は対象外）

### 1-3. claude-code-setup（Anthropic 公式）

- **リポジトリ**: [anthropics/claude-plugins-official/plugins/claude-code-setup](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-code-setup)
- **概要**: コードベースを分析し、MCP Servers / Skills / Hooks / Subagents / Plugins の中から最適な自動化を推薦
- **対象**: 推薦のみ（実際のプロンプト作成・レビューは行わない）

---

## 2. 各ツールの性質比較

| 観点 | prompt-smith | skill-creator（公式） | claude-md-management（公式） | claude-code-setup（公式） |
|------|-------------|---------------------|---------------------------|-------------------------|
| **対象コンポーネント** | Skill, Agent, Command | Skill のみ | CLAUDE.md | 全般（推薦のみ） |
| **作成機能** | ✅ 3種すべて対応 | ✅ Skill のみ | ❌ | ❌ |
| **レビュー機能** | ✅ バッチ/一貫性レビュー | ❌（Eval で代替） | ✅ CLAUDE.md の品質監査 | ❌ |
| **テスト/評価** | ❌ | ✅ Eval + Benchmark | ❌ | ❌ |
| **パターン分析** | ✅ 公式リポジトリの統計分析に基づく | ❌（ベストプラクティスは内包） | ❌ | ❌ |
| **反復改善ループ** | ❌ | ✅ Create → Eval → Improve サイクル | ❌ | ❌ |
| **description 最適化** | ❌ | ✅ トリガー精度の自動最適化 | ❌ | ❌ |
| **ベンチマーク** | ❌ | ✅ 定量的比較（pass_rate, tokens, duration） | ❌ | ❌ |
| **ブラウザUI** | ❌ | ✅ eval-viewer | ❌ | ❌ |
| **Agent 作成支援** | ✅ モデル選択/ツール/色/例ブロック | ❌ | ❌ | ❌ |
| **Command 作成支援** | ✅ 複雑度パターン/フェーズ構造 | ❌ | ❌ | ❌ |
| **プロンプトの静的レビュー** | ✅ A/B/C/D 評価 | ❌ | ❌ | ❌ |
| **作者** | 個人（naito） | Anthropic 公式 | Anthropic 公式 | Anthropic 公式 |

---

## 3. メリット・デメリットと使い分け

### prompt-smith

**メリット**:
- **3種すべてをカバー**: Skill / Agent / Command の作成・レビューを一貫して扱える唯一のツール
- **パターン分析に基づく**: 公式リポジトリ 25 Skills / 31 Agents / 41 Commands の統計分析から導出したベストプラクティス
- **静的レビュー**: 既存プロンプトの品質を A〜D で即座に評価でき、バッチ処理や一貫性チェックも可能
- **軽量**: テスト実行やベンチマークの仕組みが不要で、すぐに使い始められる

**デメリット**:
- **テスト/評価機能がない**: 作成したプロンプトが実際に意図通り動作するかの検証手段を持たない
- **description 最適化がない**: スキルのトリガー精度を自動で改善する仕組みがない
- **反復改善ループがない**: 「作成 → テスト → フィードバック → 改善」のサイクルが組み込まれていない
- **個人開発**: 公式のサポートや継続的なアップデートの保証がない

### skill-creator（公式）

**メリット**:
- **テスト駆動**: 作成 → Eval → 改善の反復ループが組み込まれており、品質を定量的に担保できる
- **description 最適化**: トリガー精度をtrain/test分割で自動最適化する専用スクリプト
- **ベンチマーク**: baseline 比較による定量的な改善効果の計測
- **ブラウザベースのレビューUI**: 結果の視覚的な確認とフィードバック収集
- **公式サポート**: Anthropic が開発・メンテナンス

**デメリット**:
- **Skill のみ対象**: Agent や Command の作成・レビューには使えない
- **重量級**: テスト実行にサブエージェントの並列起動が必要で、トークンコストが大きい
- **静的レビュー機能がない**: 既存プロンプトの品質チェックのみを行う軽量な手段がない
- **学習コスト**: 4つのモードと複雑なワークフローを理解する必要がある

### claude-md-management（公式）

**メリット**: CLAUDE.md に特化した品質監査と改善提案が充実
**デメリット**: Skill/Agent/Command プロンプトは対象外

### claude-code-setup（公式）

**メリット**: コードベースに最適な自動化を推薦してくれる（入口として有用）
**デメリット**: 推薦のみで、実際のプロンプト作成・レビューは行わない

---

## 使い分けの方針

```
┌─────────────────────────────────────────────────┐
│ 「どの自動化が必要か分からない」                    │
│  → claude-code-setup で推薦を受ける               │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 「Skill を本格的に作りたい・品質を担保したい」       │
│  → skill-creator（テスト駆動 + ベンチマーク）      │
│                                                   │
│ 「Agent / Command を作りたい」                     │
│  → prompt-smith（唯一の対応ツール）                │
│                                                   │
│ 「既存プロンプトの品質を素早くチェックしたい」        │
│  → prompt-smith（静的レビュー + バッチ処理）        │
│                                                   │
│ 「CLAUDE.md を整理したい」                         │
│  → claude-md-management                          │
└─────────────────────────────────────────────────┘
```

### 推奨: 併用パターン

| シナリオ | 使うツール |
|---------|-----------|
| 新規 Skill を高品質に作りたい | skill-creator で作成・テスト → prompt-smith でレビュー |
| 新規 Agent / Command を作りたい | prompt-smith で作成・レビュー |
| 既存プロンプト群の品質監査 | prompt-smith（バッチレビュー） |
| Skill の description を最適化したい | skill-creator（description optimization） |
| プロジェクトの CLAUDE.md 改善 | claude-md-management |
| 何から始めるべきか分からない | claude-code-setup → 各ツール |

### 今後の prompt-smith の方向性について

skill-creator は Skill 作成において圧倒的に成熟しているため、prompt-smith が Skill 作成で差別化するのは困難。
一方で以下は prompt-smith の独自価値であり、強化の方向として有望：

1. **Agent / Command の作成支援** — 公式には存在しない領域
2. **静的レビュー（バッチ/一貫性チェック）** — skill-creator にはないライトウェイトな品質チェック
3. **パターン分析の知識ベース** — 公式リポジトリの統計に基づくベストプラクティスの体系化
4. **skill-creator との連携** — skill-creator で作成した Skill を prompt-smith でレビューする補完関係
