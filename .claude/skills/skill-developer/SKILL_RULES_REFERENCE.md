# skill-rules.json - 完全リファレンス

`.claude/skills/skill-rules.json`の完全スキーマおよび設定リファレンスです。

## 目次

- [ファイル場所](#ファイル場所)
- [完全TypeScriptスキーマ](#完全typescriptスキーマ)
- [フィールドガイド](#フィールドガイド)
- [例: Guardrail Skill](#例-guardrail-skill)
- [例: Domain Skill](#例-domain-skill)
- [検証](#検証)

---

## ファイル場所

**パス:** `.claude/skills/skill-rules.json`

このJSONファイルは自動活性化システムのためのすべてのskillsとトリガー条件を定義します。

---

## 完全TypeScriptスキーマ

```typescript
interface SkillRules {
    version: string;
    skills: Record<string, SkillRule>;
}

interface SkillRule {
    type: 'guardrail' | 'domain';
    enforcement: 'block' | 'suggest' | 'warn';
    priority: 'critical' | 'high' | 'medium' | 'low';

    promptTriggers?: {
        keywords?: string[];
        intentPatterns?: string[];  // Regex文字列
    };

    fileTriggers?: {
        pathPatterns: string[];     // Globパターン
        pathExclusions?: string[];  // Globパターン
        contentPatterns?: string[]; // Regex文字列
        createOnly?: boolean;       // ファイル作成時のみトリガー
    };

    blockMessage?: string;  // Guardrails用、{file_path}プレースホルダー

    skipConditions?: {
        sessionSkillUsed?: boolean;      // セッションで使用された場合スキップ
        fileMarkers?: string[];          // 例: ["@skip-validation"]
        envOverride?: string;            // 例: "SKIP_DB_VERIFICATION"
    };
}
```

---

## フィールドガイド

### トップレベル

| フィールド | タイプ | 必須 | 説明 |
|------|------|------|------|
| `version` | string | はい | スキーマバージョン（現在"1.0"） |
| `skills` | object | はい | skill名 → SkillRuleマップ |

### SkillRuleフィールド

| フィールド | タイプ | 必須 | 説明 |
|------|------|------|------|
| `type` | string | はい | "guardrail"（強制）または"domain"（推奨） |
| `enforcement` | string | はい | "block"（PreToolUse）、"suggest"（UserPromptSubmit）、または"warn" |
| `priority` | string | はい | "critical"、"high"、"medium"、または"low" |
| `promptTriggers` | object | 任意 | UserPromptSubmit hook用トリガー |
| `fileTriggers` | object | 任意 | PreToolUse hook用トリガー |
| `blockMessage` | string | 任意* | enforcement="block"の場合は必須。`{file_path}`プレースホルダーを使用 |
| `skipConditions` | object | 任意 | 脱出条件とセッショントラッキング |

*Guardrailsには必須

### promptTriggersフィールド

| フィールド | タイプ | 必須 | 説明 |
|------|------|------|------|
| `keywords` | string[] | 任意 | 正確な部分文字列マッチング（大文字小文字を区別しない） |
| `intentPatterns` | string[] | 任意 | Intent検出用Regexパターン |

### fileTriggersフィールド

| フィールド | タイプ | 必須 | 説明 |
|------|------|------|------|
| `pathPatterns` | string[] | はい* | ファイルパス用Globパターン |
| `pathExclusions` | string[] | 任意 | 除外するGlobパターン（例: テストファイル） |
| `contentPatterns` | string[] | 任意 | ファイル内容マッチング用Regexパターン |
| `createOnly` | boolean | 任意 | 新規ファイル作成時のみトリガー |

*fileTriggersがある場合は必須

### skipConditionsフィールド

| フィールド | タイプ | 必須 | 説明 |
|------|------|------|------|
| `sessionSkillUsed` | boolean | 任意 | このセッションでskillが既に使用された場合スキップ |
| `fileMarkers` | string[] | 任意 | ファイルにコメントマーカーが含まれる場合スキップ |
| `envOverride` | string | 任意 | Skillを無効化する環境変数名 |

---

## 例: Guardrail Skill

すべての機能を含むブロックguardrail skillの完全な例:

```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",

    "promptTriggers": {
      "keywords": [
        "prisma",
        "database",
        "table",
        "column",
        "schema",
        "query",
        "migration"
      ],
      "intentPatterns": [
        "(add|create|implement).*?(user|login|auth|tracking|feature)",
        "(modify|update|change).*?(table|column|schema|field)",
        "database.*?(change|update|modify|migration)"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "**/schema.prisma",
        "**/migrations/**/*.sql",
        "database/src/**/*.ts",
        "form/src/**/*.ts",
        "email/src/**/*.ts",
        "users/src/**/*.ts",
        "projects/src/**/*.ts",
        "utilities/src/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts",
        "**/*.spec.ts"
      ],
      "contentPatterns": [
        "import.*[Pp]risma",
        "PrismaService",
        "prisma\\.",
        "\\.findMany\\(",
        "\\.findUnique\\(",
        "\\.findFirst\\(",
        "\\.create\\(",
        "\\.createMany\\(",
        "\\.update\\(",
        "\\.updateMany\\(",
        "\\.upsert\\(",
        "\\.delete\\(",
        "\\.deleteMany\\("
      ]
    },

    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED ACTION:\n1. Use Skill tool: 'database-verification'\n2. Verify ALL table and column names against schema\n3. Check database structure with DESCRIBE commands\n4. Then retry this edit\n\nReason: Prevent column name errors in Prisma queries\nFile: {file_path}\n\n💡 TIP: Add '// @skip-validation' comment to skip future checks",

    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": [
        "@skip-validation"
      ],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

### Guardrails重要ポイント

1. **type**: "guardrail"である必要がある
2. **enforcement**: "block"である必要がある
3. **priority**: 通常"critical"または"high"
4. **blockMessage**: 必須、明確で実行可能なステップ
5. **skipConditions**: セッショントラッキングで繰り返し通知を防止
6. **fileTriggers**: 通常パスとコンテンツパターンの両方を含む
7. **contentPatterns**: 技術の実際の使用を検出

---

## 例: Domain Skill

提案ベースのdomain skillの完全な例:

```json
{
  "project-catalog-developer": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",

    "promptTriggers": {
      "keywords": [
        "layout",
        "layout system",
        "grid",
        "grid layout",
        "toolbar",
        "column",
        "cell editor",
        "cell renderer",
        "submission",
        "submissions",
        "blog dashboard",
        "datagrid",
        "data grid",
        "CustomToolbar",
        "GridLayoutDialog",
        "useGridLayout",
        "auto-save",
        "column order",
        "column width",
        "filter",
        "sort"
      ],
      "intentPatterns": [
        "(how does|how do|explain|what is|describe).*?(layout|grid|toolbar|column|submission|catalog)",
        "(add|create|modify|change).*?(toolbar|column|cell|editor|renderer)",
        "blog dashboard.*?"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/features/submissions/**/*.tsx",
        "frontend/src/features/submissions/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.tsx",
        "**/*.test.ts"
      ]
    }
  }
}
```

### Domain Skills重要ポイント

1. **type**: "domain"である必要がある
2. **enforcement**: 通常"suggest"
3. **priority**: "high"または"medium"
4. **blockMessage**: 不要（ブロックしない）
5. **skipConditions**: 任意（重要度が低い）
6. **promptTriggers**: 通常広範なキーワードを含む
7. **fileTriggers**: パスパターンのみの場合がある（コンテンツは重要度が低い）

---

## 検証

### JSON構文の確認

```bash
cat .claude/skills/skill-rules.json | jq .
```

有効な場合、jqがJSONを整形して出力します。無効な場合はエラーを表示します。

### 一般的なJSONエラー

**末尾のカンマ:**
```json
{
  "keywords": ["one", "two",]  // ❌ 末尾のカンマ
}
```

**引用符の欠落:**
```json
{
  type: "guardrail"  // ❌ キーに引用符が欠落
}
```

**シングルクォート（無効なJSON）:**
```json
{
  'type': 'guardrail'  // ❌ ダブルクォートを使用すべき
}
```

### 検証チェックリスト

- [ ] JSON構文が有効（`jq`を使用）
- [ ] すべてのskill名がSKILL.mdファイル名と一致
- [ ] Guardrailsに`blockMessage`がある
- [ ] ブロックメッセージが`{file_path}`プレースホルダーを使用
- [ ] Intentパターンが有効なregex（regex101.comでテスト）
- [ ] ファイルパスパターンが正しいglob構文を使用
- [ ] コンテンツパターンが特殊文字をエスケープ
- [ ] 優先度が適用レベルと一致
- [ ] 重複するskill名がない

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインskillガイド
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 完全トリガードキュメント
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - 設定問題のデバッグ
