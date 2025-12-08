# トリガータイプ - 包括的ガイド

Claude Codeのskill自動活性化システムでskillトリガーを設定するための包括的な参照ドキュメントです。

## 目次

- [キーワードトリガー（明示的）](#キーワードトリガー明示的)
- [Intentパターントリガー（暗黙的）](#intentパターントリガー暗黙的)
- [ファイルパストリガー](#ファイルパストリガー)
- [コンテンツパターントリガー](#コンテンツパターントリガー)
- [ベストプラクティスサマリー](#ベストプラクティスサマリー)

---

## キーワードトリガー（明示的）

### 動作方式

ユーザープロンプトで大文字小文字を区別しない部分文字列マッチングを実行します。

### 用途

ユーザーがトピックを明示的に言及するトピックベースの活性化に使用します。

### 設定

```json
"promptTriggers": {
  "keywords": ["layout", "grid", "toolbar", "submission"]
}
```

### 例

- ユーザープロンプト: "**layout**システムはどのように動作しますか？"
- マッチ: "layout"キーワード
- 活性化: `project-catalog-developer`

### ベストプラクティス

- 具体的で曖昧でない用語を使用
- 一般的なバリエーションを含める（"layout", "layout system", "grid layout"）
- あまりにも一般的な単語を避ける（"system", "work", "create"）
- 実際のプロンプトでテスト

---

## Intentパターントリガー（暗黙的）

### 動作方式

ユーザーがトピックを明示的に言及しなくても意図を検出するためのregexパターンマッチングです。

### 用途

ユーザーが特定のトピックではなく望む作業を説明する動作ベースの活性化に使用します。

### 設定

```json
"promptTriggers": {
  "intentPatterns": [
    "(create|add|implement).*?(feature|endpoint)",
    "(how does|explain).*?(layout|workflow)"
  ]
}
```

### 例

**データベース作業:**
- ユーザープロンプト: "ユーザートラッキング機能を追加して"
- マッチ: `(add).*?(feature)`
- 活性化: `database-verification`, `error-tracking`

**コンポーネント作成:**
- ユーザープロンプト: "ダッシュボードウィジェットを作成して"
- マッチ: `(create).*?(component)`（パターンにcomponentがある場合）
- 活性化: `frontend-dev-guidelines`

### ベストプラクティス

- 一般的な動作動詞を捕捉: `(create|add|modify|build|implement)`
- ドメイン固有の名詞を含める: `(feature|endpoint|component|workflow)`
- 非貪欲マッチングを使用: `.*`の代わりに`.*?`
- regexテスターでパターンを徹底的にテスト（https://regex101.com/）
- パターンを広すぎないように（誤検知発生）
- パターンを具体的すぎないように（検出漏れ発生）

### 一般的なパターン例

```regex
# データベース作業
(add|create|implement).*?(user|login|auth|feature)

# 説明リクエスト
(how does|explain|what is|describe).*?

# フロントエンド作業
(create|add|make|build).*?(component|UI|page|modal|dialog)

# エラー処理
(fix|handle|catch|debug).*?(error|exception|bug)

# ワークフロー作業
(create|add|modify).*?(workflow|step|branch|condition)
```

---

## ファイルパストリガー

### 動作方式

編集中のファイルパスに対してglobパターンマッチングを実行します。

### 用途

プロジェクト内のファイル位置に基づくドメイン/領域別の活性化に使用します。

### 設定

```json
"fileTriggers": {
  "pathPatterns": [
    "frontend/src/**/*.tsx",
    "form/src/**/*.ts"
  ],
  "pathExclusions": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

### Globパターン構文

- `**` = 複数のディレクトリ（0個を含む）
- `*` = ディレクトリ名内の任意の文字
- 例:
  - `frontend/src/**/*.tsx` = frontend/srcとサブディレクトリのすべての.tsxファイル
  - `**/schema.prisma` = プロジェクトのどこでもschema.prisma
  - `form/src/**/*.ts` = form/srcのサブディレクトリのすべての.tsファイル

### 例

- 編集中のファイル: `frontend/src/components/Dashboard.tsx`
- マッチ: `frontend/src/**/*.tsx`
- 活性化: `frontend-dev-guidelines`

### ベストプラクティス

- 誤検知を防ぐために具体的に記述
- テストファイルの除外を使用: `**/*.test.ts`
- サブディレクトリ構造を考慮
- 実際のファイルパスでパターンをテスト
- 可能な場合はより狭いパターンを使用: `form/**`の代わりに`form/src/services/**`

### 一般的なパスパターン

```glob
# フロントエンド
frontend/src/**/*.tsx        # すべてのReactコンポーネント
frontend/src/**/*.ts         # すべてのTypeScriptファイル
frontend/src/components/**   # componentsディレクトリのみ

# バックエンドサービス
form/src/**/*.ts            # Formサービス
email/src/**/*.ts           # Emailサービス
users/src/**/*.ts           # Usersサービス

# データベース
**/schema.prisma            # Prismaスキーマ（どこでも）
**/migrations/**/*.sql      # マイグレーションファイル
database/src/**/*.ts        # データベーススクリプト

# ワークフロー
form/src/workflow/**/*.ts              # ワークフローエンジン
form/src/workflow-definitions/**/*.json # ワークフロー定義

# テスト除外
**/*.test.ts                # TypeScriptテスト
**/*.test.tsx               # Reactコンポーネントテスト
**/*.spec.ts                # Specファイル
```

---

## コンテンツパターントリガー

### 動作方式

ファイルの実際の内容（ファイルの中にあるもの）に対してregexパターンマッチングを実行します。

### 用途

コードがimportまたは使用しているもの（Prisma、コントローラー、特定のライブラリ）に基づく技術特化の活性化に使用します。

### 設定

```json
"fileTriggers": {
  "contentPatterns": [
    "import.*[Pp]risma",
    "PrismaService",
    "\\.findMany\\(",
    "\\.create\\("
  ]
}
```

### 例

**Prisma検出:**
- ファイル内容: `import { PrismaService } from '@project/database'`
- マッチ: `import.*[Pp]risma`
- 活性化: `database-verification`

**Controller検出:**
- ファイル内容: `export class UserController {`
- マッチ: `export class.*Controller`
- 活性化: `error-tracking`

### ベストプラクティス

- importマッチング: `import.*[Pp]risma`（[Pp]で大文字小文字を区別しない）
- 特殊regex文字をエスケープ: `.findMany(`の代わりに`\\.findMany\\(`
- パターンは大文字小文字を区別しないフラグを使用
- 実際のファイル内容でテスト
- 誤ったマッチングを避けるために十分に具体的に記述

### 一般的なコンテンツパターン

```regex
# Prisma/データベース
import.*[Pp]risma                # Prisma import
PrismaService                    # PrismaService使用
prisma\.                         # prisma.something
\.findMany\(                     # Prismaクエリメソッド
\.create\(
\.update\(
\.delete\(

# コントローラー/ルート
export class.*Controller         # Controllerクラス
router\.                         # Express router
app\.(get|post|put|delete|patch) # Express appルート

# エラー処理
try\s*\{                        # Tryブロック
catch\s*\(                      # Catchブロック
throw new                        # Throw文

# React/コンポーネント
export.*React\.FC               # React関数コンポーネント
export default function.*       # デフォルト関数export
useState|useEffect              # React hooks
```

---

## ベストプラクティスサマリー

### すべきこと:
✅ 具体的で曖昧でないキーワードを使用
✅ すべてのパターンを実際の例でテスト
✅ 一般的なバリエーションを含める
✅ 非貪欲regex使用: `.*?`
✅ コンテンツパターンで特殊文字をエスケープ
✅ テストファイルの除外を追加
✅ ファイルパスパターンを狭く具体的に記述

### すべきでないこと:
❌ あまりにも一般的なキーワードを使用（"system", "work"）
❌ Intentパターンを広すぎるように作成（誤検知）
❌ パターンを具体的すぎるように作成（検出漏れ）
❌ regexテスターでテストしない（https://regex101.com/）
❌ `.*?`の代わりに貪欲regex`.*`を使用
❌ ファイルパスで広すぎるマッチング

### トリガーのテスト

**キーワード/intentトリガーのテスト:**
```bash
echo '{"session_id":"test","prompt":"テストプロンプト"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**ファイルパス/コンテンツトリガーのテスト:**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/path/to/test/file.ts"}
}
EOF
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインskillガイド
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 完全skill-rules.jsonスキーマ
- [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md) - すぐに使えるパターンライブラリ
