---
name: skill-developer
description: Create and manage Claude Code skills following Anthropic best practices. Use when creating new skills, modifying skill-rules.json, understanding trigger patterns, working with hooks, debugging skill activation, or implementing progressive disclosure. Covers skill structure, YAML frontmatter, trigger types (keywords, intent patterns, file paths, content patterns), enforcement levels (block, suggest, warn), hook mechanisms (UserPromptSubmit, PreToolUse), session tracking, and the 500-line rule.
---

# Skill 開発者ガイド

## 目的

Anthropicの公式ベストプラクティス（500-line rule、Progressive disclosureパターンを含む）に従ったClaude Code skillsの作成と管理についての包括的なガイドです。

## このSkillの使用タイミング

以下を言及した場合に自動活性化されます：
- Skillsの作成または追加
- Skill triggersまたはrulesの修正
- Skill活性化の動作の理解
- Skill活性化問題のデバッグ
- skill-rules.jsonの作業
- Hookシステムメカニズム
- Claude Codeベストプラクティス
- Progressive disclosure
- YAML frontmatter
- 500-line rule

---

## システム概要

### 2-Hookアーキテクチャ

**1. UserPromptSubmit Hook**（事前提案）
- **ファイル**: `.claude/hooks/skill-activation-prompt.ts`
- **トリガー**: Claudeがユーザープロンプトを見る前に
- **目的**: キーワード + intentパターンに基づいて関連skillsを提案
- **方式**: フォーマットされたリマインダーをcontextに注入（stdout → Claude入力）
- **使用事例**: トピックベースのskills、暗黙的なタスク検出

**2. Stop Hook - エラー処理リマインダー**（ソフトな通知）
- **ファイル**: `.claude/hooks/error-handling-reminder.ts`
- **トリガー**: Claudeが応答を完了した後に
- **目的**: 作成したコードのエラー処理についての自己評価のためのソフトなリマインダー
- **方式**: 編集されたファイルで危険なパターンを分析し、必要に応じてリマインダーを表示
- **使用事例**: ブロック摩擦なしにエラー処理の意識を維持

**哲学変更（2025-10-27）：** Sentry/エラー処理のためのブロックPreToolUseから離れました。代わりに、ワークフローをブロックせずにコード品質の意識を維持するソフトな応答後リマインダーを使用します。

### 設定ファイル

**場所**: `.claude/skills/skill-rules.json`

定義内容:
- すべてのskillsとトリガー条件
- 適用レベル（block, suggest, warn）
- ファイルパスパターン（glob）
- コンテンツ検出パターン（regex）
- スキップ条件（セッショントラッキング、ファイルマーカー、環境変数）

---

## Skillタイプ

### 1. Guardrail Skills

**目的:** エラーを防ぐコアベストプラクティスの強制

**特性:**
- Type: `"guardrail"`
- Enforcement: `"block"`
- Priority: `"critical"` または `"high"`
- Skill使用前までファイル編集をブロック
- 一般的なミスの防止（カラム名、致命的エラー）
- セッション認識（同一セッションでの繰り返し通知なし）

**例:**
- `database-verification` - Prismaクエリ前のテーブル/カラム名検証
- `frontend-dev-guidelines` - React/TypeScriptパターンの強制

**使用タイミング:**
- ランタイムエラーを引き起こすミス
- データ整合性関連の問題
- 致命的な互換性問題

### 2. Domain Skills

**目的:** 特定領域についての包括的なガイドの提供

**特性:**
- Type: `"domain"`
- Enforcement: `"suggest"`
- Priority: `"high"` または `"medium"`
- 推奨事項、強制ではない
- トピックまたはドメイン特化
- 包括的なドキュメント

**例:**
- `backend-dev-guidelines` - Node.js/Express/TypeScriptパターン
- `frontend-dev-guidelines` - React/TypeScriptベストプラクティス
- `error-tracking` - Sentry統合ガイド

**使用タイミング:**
- 深い知識が必要な複雑なシステム
- ベストプラクティスのドキュメント
- アーキテクチャパターン
- How-toガイド

---

## クイックスタート: 新しいSkillの作成

### ステップ1: Skillファイルの作成

**場所:** `.claude/skills/{skill-name}/SKILL.md`

**テンプレート:**
```markdown
---
name: my-new-skill
description: このskillをトリガーするキーワードを含む簡単な説明。トピック、ファイルタイプ、使用事例を言及してください。トリガー用語を明示的に記載してください。
---

# 私の新しいSkill

## 目的
このskillが役立つ場面

## 使用タイミング
特定のシナリオと条件

## コア情報
実際のガイド、ドキュメント、パターン、例
```

**ベストプラクティス:**
- ✅ **名前**: 小文字、ハイフン、動名詞形式を推奨
- ✅ **説明**: すべてのトリガーキーワード/フレーズを含む（最大1024文字）
- ✅ **内容**: 500行未満 - 詳細情報は参照ファイルを使用
- ✅ **例**: 実際のコード例
- ✅ **構造**: 明確な見出し、リスト、コードブロック

### ステップ2: skill-rules.jsonへの追加

全体スキーマは[SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)を参照してください。

**基本テンプレート:**
```json
{
  "my-new-skill": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "medium",
    "promptTriggers": {
      "keywords": ["keyword1", "keyword2"],
      "intentPatterns": ["(create|add).*?something"]
    }
  }
}
```

### ステップ3: トリガーのテスト

**UserPromptSubmitテスト:**
```bash
echo '{"session_id":"test","prompt":"テストプロンプト"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**PreToolUseテスト:**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

### ステップ4: パターンの改善

テスト結果に基づいて:
- 不足しているキーワードを追加
- 誤検知を減らすためにintentパターンを改善
- ファイルパスパターンを調整
- 実際のファイルでコンテンツパターンをテスト

### ステップ5: Anthropicベストプラクティスの遵守

✅ SKILL.mdを500行未満に維持
✅ 参照ファイルでProgressive disclosureを使用
✅ 100行を超える参照ファイルには目次を追加
✅ トリガーキーワードを含む詳細な説明を作成
✅ ドキュメント化前に3つ以上の実際のシナリオでテスト
✅ 実際の使用に基づいて反復改善

---

## 適用レベル

### BLOCK（コアGuardrails）

- Edit/Writeツール実行を物理的に防止
- Hookから終了コード2、stderr → Claude
- Claudeがメッセージを見てskillを使用すれば進行可能
- **使用対象**: 致命的なミス、データ整合性、セキュリティ問題

**例:** データベースカラム名の検証

### SUGGEST（推奨）

- Claudeがプロンプトを見る前にリマインダーを注入
- Claudeが関連skillsを認識
- 強制ではなく、推奨のみ
- **使用対象**: ドメインガイド、ベストプラクティス、how-toガイド

**例:** フロントエンド開発ガイドライン

### WARN（オプション）

- 低優先度の提案
- 推奨のみ、最小限の適用
- **使用対象**: あると良い提案、情報提供のリマインダー

**まれに使用** - ほとんどのskillsはBLOCKまたはSUGGESTです。

---

## スキップ条件とユーザー制御

### 1. セッショントラッキング

**目的:** 同一セッションでの繰り返し通知の防止

**動作方式:**
- 最初の編集 → Hookがブロックしセッション状態を更新
- 2回目の編集（同一セッション） → Hookが許可
- 別のセッション → 再度ブロック

**状態ファイル:** `.claude/hooks/state/skills-used-{session_id}.json`

### 2. ファイルマーカー

**目的:** 検証済みファイルの永久スキップ

**マーカー:** `// @skip-validation`

**使用法:**
```typescript
// @skip-validation
import { PrismaService } from './prisma';
// このファイルは手動で検証されました
```

**注意:** 乱用すると目的が無効になるため慎重に使用

### 3. 環境変数

**目的:** 緊急無効化、一時的なオーバーライド

**グローバル無効化:**
```bash
export SKIP_SKILL_GUARDRAILS=true  # すべてのPreToolUseブロックを無効化
```

**Skill別:**
```bash
export SKIP_DB_VERIFICATION=true
export SKIP_ERROR_REMINDER=true
```

---

## テストチェックリスト

新しいskill作成時の確認事項:

- [ ] `.claude/skills/{name}/SKILL.md`にskillファイルが作成された
- [ ] 名前と説明を含む正しいfrontmatter
- [ ] `skill-rules.json`にエントリが追加された
- [ ] 実際のプロンプトでキーワードがテストされた
- [ ] 様々なバリエーションでintentパターンがテストされた
- [ ] 実際のファイルでファイルパスパターンがテストされた
- [ ] ファイル内容でコンテンツパターンがテストされた
- [ ] ブロックメッセージが明確で実行可能（guardrailの場合）
- [ ] スキップ条件が適切に設定された
- [ ] 優先度レベルが重要度と一致している
- [ ] テストで誤検知なし
- [ ] テストで検出漏れなし
- [ ] パフォーマンスが許容範囲内（<100msまたは<200ms）
- [ ] JSON構文の検証: `jq . skill-rules.json`
- [ ] **SKILL.mdが500行未満** ⭐
- [ ] 必要に応じて参照ファイルが作成された
- [ ] 100行を超えるファイルには目次が追加された

---

## 参照ファイル

特定のトピックについての詳細情報は以下を参照してください:

### [TRIGGER_TYPES.md](TRIGGER_TYPES.md)
すべてのトリガータイプの包括的なガイド:
- キーワードトリガー（明示的トピックマッチング）
- Intentパターン（暗黙的動作検出）
- ファイルパストリガー（globパターン）
- コンテンツパターン（ファイル内regex）
- 各タイプのベストプラクティスと例
- 一般的な落とし穴とテスト戦略

### [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)
skill-rules.jsonの完全スキーマ:
- 完全なTypeScriptインターフェース定義
- フィールド別の説明
- 完全なguardrail skillの例
- 完全なdomain skillの例
- 検証ガイドと一般的なエラー

### [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md)
Hook内部の深堀り分析:
- UserPromptSubmitフロー（詳細）
- PreToolUseフロー（詳細）
- 終了コード動作表（重要）
- セッション状態管理
- パフォーマンス考慮事項

### [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
包括的なデバッグガイド:
- Skillがトリガーされない（UserPromptSubmit）
- PreToolUseがブロックしない
- 誤検知（トリガーが多すぎる）
- Hookが全く実行されない
- パフォーマンス問題

### [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md)
すぐに使えるパターンコレクション:
- Intentパターンライブラリ（regex）
- ファイルパスパターンライブラリ（glob）
- コンテンツパターンライブラリ（regex）
- 使用事例別整理
- コピー＆ペースト可能

### [ADVANCED.md](ADVANCED.md)
今後の改善事項とアイデア:
- 動的ルール更新
- Skill依存関係
- 条件付き適用
- Skill分析
- Skillバージョン管理

---

## クイックリファレンスサマリー

### 新規Skill作成（5ステップ）

1. frontmatterとともに`.claude/skills/{name}/SKILL.md`を作成
2. `.claude/skills/skill-rules.json`にエントリを追加
3. `npx tsx`コマンドでテスト
4. テストに基づいてパターンを改善
5. SKILL.mdを500行未満に維持

### トリガータイプ

- **Keywords**: 明示的なトピック言及
- **Intent**: 暗黙的な動作検出
- **File Paths**: 場所ベースの活性化
- **Content**: 技術特化の検出

詳細は[TRIGGER_TYPES.md](TRIGGER_TYPES.md)を参照してください。

### 適用

- **BLOCK**: 終了コード2、コア事項のみ
- **SUGGEST**: context注入、最も一般的
- **WARN**: 推奨、まれに使用

### スキップ条件

- **セッショントラッキング**: 自動（繰り返し通知防止）
- **ファイルマーカー**: `// @skip-validation`（永久スキップ）
- **環境変数**: `SKIP_SKILL_GUARDRAILS`（緊急無効化）

### Anthropicベストプラクティス

✅ **500-line rule**: SKILL.mdを500行未満に維持
✅ **Progressive disclosure**: 詳細情報は参照ファイルを使用
✅ **目次**: 100行を超える参照ファイルに追加
✅ **1段階の深さ**: 参照を深くネストしない
✅ **豊富な説明**: すべてのトリガーキーワードを含む（最大1024文字）
✅ **まずテスト**: 広範なドキュメント化前に3つ以上の評価を構築
✅ **動名詞命名**: 動詞 + -ingを推奨（例: "processing-pdfs"）

### トラブルシューティング

手動でhooksをテスト:
```bash
# UserPromptSubmit
echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

完全なデバッグガイドは[TROUBLESHOOTING.md](TROUBLESHOOTING.md)を参照してください。

---

## 関連ファイル

**設定:**
- `.claude/skills/skill-rules.json` - マスター設定
- `.claude/hooks/state/` - セッショントラッキング
- `.claude/settings.json` - Hook登録

**Hooks:**
- `.claude/hooks/skill-activation-prompt.ts` - UserPromptSubmit
- `.claude/hooks/error-handling-reminder.ts` - Stopイベント（ソフトリマインダー）

**すべてのSkills:**
- `.claude/skills/*/SKILL.md` - Skillコンテンツファイル

---

**Skill状態**: COMPLETE - Anthropicベストプラクティスに従って再構成済み ✅
**行数**: < 500（500-line rule遵守） ✅
**Progressive Disclosure**: 詳細情報のための参照ファイル ✅

**次のステップ**: より多くのskillsを作成、使用に基づいてパターンを改善
