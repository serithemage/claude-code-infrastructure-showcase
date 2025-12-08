# 共通パターンライブラリ

Skillトリガーのためのすぐに使えるregexおよびglobパターンです。コピーしてカスタマイズしてください。

---

## Intentパターン（Regex）

### 機能/エンドポイント作成
```regex
(add|create|implement|build).*?(feature|endpoint|route|service|controller)
```

### コンポーネント作成
```regex
(create|add|make|build).*?(component|UI|page|modal|dialog|form)
```

### データベース作業
```regex
(add|create|modify|update).*?(user|table|column|field|schema|migration)
(database|prisma).*?(change|update|query)
```

### エラー処理
```regex
(fix|handle|catch|debug).*?(error|exception|bug)
(add|implement).*?(try|catch|error.*?handling)
```

### 説明リクエスト
```regex
(how does|how do|explain|what is|describe|tell me about).*?
```

### ワークフロー作業
```regex
(create|add|modify|update).*?(workflow|step|branch|condition)
(debug|troubleshoot|fix).*?workflow
```

### テスト
```regex
(write|create|add).*?(test|spec|unit.*?test)
```

---

## ファイルパスパターン（Glob）

### フロントエンド
```glob
frontend/src/**/*.tsx        # すべてのReactコンポーネント
frontend/src/**/*.ts         # すべてのTypeScriptファイル
frontend/src/components/**   # componentsディレクトリのみ
```

### バックエンドサービス
```glob
form/src/**/*.ts            # Formサービス
email/src/**/*.ts           # Emailサービス
users/src/**/*.ts           # Usersサービス
projects/src/**/*.ts        # Projectsサービス
```

### データベース
```glob
**/schema.prisma            # Prismaスキーマ（どこでも）
**/migrations/**/*.sql      # マイグレーションファイル
database/src/**/*.ts        # データベーススクリプト
```

### ワークフロー
```glob
form/src/workflow/**/*.ts              # ワークフローエンジン
form/src/workflow-definitions/**/*.json # ワークフロー定義
```

### テスト除外
```glob
**/*.test.ts                # TypeScriptテスト
**/*.test.tsx               # Reactコンポーネントテスト
**/*.spec.ts                # Specファイル
```

---

## コンテンツパターン（Regex）

### Prisma/データベース
```regex
import.*[Pp]risma                # Prisma import
PrismaService                    # PrismaService使用
prisma\.                         # prisma.something
\.findMany\(                     # Prismaクエリメソッド
\.create\(
\.update\(
\.delete\(
```

### コントローラー/ルート
```regex
export class.*Controller         # Controllerクラス
router\.                         # Express router
app\.(get|post|put|delete|patch) # Express appルート
```

### エラー処理
```regex
try\s*\{                        # Tryブロック
catch\s*\(                      # Catchブロック
throw new                        # Throw文
```

### React/コンポーネント
```regex
export.*React\.FC               # React関数コンポーネント
export default function.*       # デフォルト関数export
useState|useEffect              # React hooks
```

---

**使用例:**

```json
{
  "my-skill": {
    "promptTriggers": {
      "intentPatterns": [
        "(create|add|build).*?(component|UI|page)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.tsx"
      ],
      "contentPatterns": [
        "export.*React\\.FC",
        "useState|useEffect"
      ]
    }
  }
}
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインskillガイド
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 詳細トリガードキュメント
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 完全スキーマ
