# Agents

複雑なマルチステップタスクのための特化型エージェント。

---

## Agentsとは？

Agentsは特定の複雑なタスクを処理する自律的なClaudeインスタンスです。Skills（インラインガイドを提供する）とは異なり、agentsは：
- 別のサブタスクとして実行
- 最小限の監督で自律的に作業
- 特化したツールへのアクセス権を保有
- 完了時に総合レポートを返す

**核心的な利点：** Agentsは**独立型** - `.md`ファイルをコピーするだけですぐに使用可能！

---

## 使用可能なAgents（10個）

### code-architecture-reviewer
**目的：** アーキテクチャの一貫性とベストプラクティスのコードレビュー

**使用時期：**
- 新機能の実装後
- 重要な変更をマージする前
- コードのリファクタリング時
- アーキテクチャ決定の検証時

**統合：** ✅ そのままコピー

---

### code-refactor-master
**目的：** 包括的なリファクタリング計画と実行

**使用時期：**
- ファイル構造の再構成
- 大きなコンポーネントの分離
- ファイル移動後のimportパス更新
- コードの保守性改善

**統合：** ✅ そのままコピー

---

### documentation-architect
**目的：** 包括的なドキュメント作成

**使用時期：**
- 新機能のドキュメント化
- APIドキュメントの作成
- 開発者ガイドの作成
- アーキテクチャ概要の生成

**統合：** ✅ そのままコピー

---

### frontend-error-fixer
**目的：** フロントエンドエラーのデバッグと修正

**使用時期：**
- ブラウザコンソールエラー
- フロントエンドTypeScriptコンパイルエラー
- Reactエラー
- ビルド失敗

**統合：** ⚠️ スクリーンショットパス参照可能 - 必要に応じて更新

---

### plan-reviewer
**目的：** 実装前の開発計画レビュー

**使用時期：**
- 複雑な機能を開始する前
- アーキテクチャ計画の検証
- 潜在的な問題の早期識別
- アプローチについてのセカンドオピニオンが必要な時

**統合：** ✅ そのままコピー

---

### refactor-planner
**目的：** 包括的なリファクタリング戦略の策定

**使用時期：**
- コード再構成の計画
- レガシーコードの現代化
- 大きなファイルの分離
- コード構造の改善

**統合：** ✅ そのままコピー

---

### web-research-specialist
**目的：** インターネットでの技術イシュー調査

**使用時期：**
- 難解なエラーのデバッグ
- 問題解決策を探す
- ベストプラクティスの調査
- 実装アプローチの比較

**統合：** ✅ そのままコピー

---

### auth-route-tester
**目的：** 認証されたAPIエンドポイントのテスト

**使用時期：**
- JWT cookie認証を使用するルートのテスト
- エンドポイント機能の検証
- 認証問題のデバッグ

**統合：** ⚠️ JWT cookie基盤認証が必要

---

### auth-route-debugger
**目的：** 認証問題のデバッグ

**使用時期：**
- 認証失敗
- トークン問題
- cookie問題
- 権限エラー

**統合：** ⚠️ JWT cookie基盤認証が必要

---

### auto-error-resolver
**目的：** TypeScriptコンパイルエラーの自動修正

**使用時期：**
- TypeScriptエラーによるビルド失敗
- 型を壊すリファクタリング後
- 体系的なエラー解決が必要な時

**統合：** ⚠️ パス更新が必要な場合あり

---

## Agent統合方法

### 標準統合（大部分のAgents）

**Step 1: ファイルコピー**
```bash
cp showcase/.claude/agents/agent-name.md \\
   your-project/.claude/agents/
```

**Step 2: 確認（オプション）**
```bash
# ハードコードされたパスを確認
grep -n "~/git/\|/root/git/\|/Users/" your-project/.claude/agents/agent-name.md
```

**Step 3: 使用する**
Claudeにリクエスト：「[agent-name] agentを使って[タスク]してください」

以上！Agentsはすぐに動作します。

---

### カスタマイズが必要なAgents

**frontend-error-fixer:**
- スクリーンショットパス参照可能
- ユーザーに確認：「スクリーンショットをどこに保存しますか？」
- エージェントファイルのパスを更新

**auth-route-tester / auth-route-debugger:**
- JWT cookie認証が必要
- 例のサービスURLを更新
- ユーザーの認証設定に合わせてカスタマイズ

**auto-error-resolver:**
- ハードコードされたプロジェクトパスがある可能性
- `$CLAUDE_PROJECT_DIR`または相対パス使用に更新

---

## Agents vs Skills使用時期

| Agents使用時... | Skills使用時... |
|-------------------|-------------------|
| マルチステップタスクが必要 | インラインガイドが必要 |
| 複雑な分析が必要 | ベストプラクティス確認 |
| 自律的作業を好む | コントロールを維持したい |
| 明確な最終目標があるタスク | 継続的な開発作業 |
| 例：「すべてのコントローラーをレビューして」 | 例：「新しいルートを作成中」 |

**両方を一緒に使用可能：**
- Skillは開発中にパターンを提供
- Agentは完了後に結果をレビュー

---

## Agentクイックリファレンス

| Agent | 複雑度 | カスタマイズ | 認証必要 |
|-------|--------|--------------|-----------|
| code-architecture-reviewer | 中 | ✅ なし | いいえ |
| code-refactor-master | 高 | ✅ なし | いいえ |
| documentation-architect | 中 | ✅ なし | いいえ |
| frontend-error-fixer | 中 | ⚠️ スクリーンショットパス | いいえ |
| plan-reviewer | 低 | ✅ なし | いいえ |
| refactor-planner | 中 | ✅ なし | いいえ |
| web-research-specialist | 低 | ✅ なし | いいえ |
| auth-route-tester | 中 | ⚠️ 認証設定 | JWT cookie |
| auth-route-debugger | 中 | ⚠️ 認証設定 | JWT cookie |
| auto-error-resolver | 低 | ⚠️ パス | いいえ |

---

## Claude Code向けガイド

**ユーザーのためにagents統合時：**

1. **[CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を読む**
2. **.mdファイルのみコピー** - agentsは独立型
3. **ハードコードされたパスを確認：**
   ```bash
   grep "~/git/\|/root/" agent-name.md
   ```
4. **パスが見つかった場合は更新** - `$CLAUDE_PROJECT_DIR`または`.`に
5. **認証agentsの場合：** JWT cookie認証使用の有無を先に確認

**以上！** Agentsは最も簡単に統合できるコンポーネントです。

---

## 独自のAgentsを作成

Agentsはオプションのyaml frontmatterがあるマークダウンファイルです：

```markdown
# Agent Name

## 目的
このエージェントが行うこと

## 指示
自律実行のためのステップバイステップの指示

## 使用可能なツール
このエージェントが使用できるツールリスト

## 予想される出力
結果を返す形式
```

**ヒント：**
- 指示を非常に具体的に記述
- 複雑なタスクを番号付きステップに分離
- 返す内容を正確に明示
- 良い出力の例を含める
- 使用可能なツールを明示的にリスト

---

## トラブルシューティング

### Agentが見つからない

**確認：**
```bash
# エージェントファイルがあるか？
ls -la .claude/agents/[agent-name].md
```

### Agentがパスエラーで失敗

**ハードコードされたパスを確認：**
```bash
grep "~/\|/root/\|/Users/" .claude/agents/[agent-name].md
```

**修正：**
```bash
sed -i 's|~/git/.*project|$CLAUDE_PROJECT_DIR|g' .claude/agents/[agent-name].md
```

---

## 次のステップ

1. **上記のagentsを見る** - 作業に役立つものを探す
2. **必要なものをコピー** - .mdファイルのみ
3. **Claudeに使用を依頼** -「[agent]を使って[タスク]してください」
4. **独自のものを作成** - 特定の要件に合わせてパターンを踏襲

**質問がありますか？** [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を参照
