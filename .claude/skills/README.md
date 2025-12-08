# Skills

Contextに基づいて自動アクティベートされる本番検証済みのClaude Code skillsです。

---

## Skillsとは？

Skillsは、Claudeが必要な時にロードするモジュラーナレッジベースです。以下を提供します：
- ドメイン別ガイドライン
- ベストプラクティス
- コード例
- 避けるべきアンチパターン

**問題：** Skillsはデフォルトで自動アクティベートされません。

**解決策：** このshowcaseはskillsをアクティベートするhooks + 設定を含みます。

---

## 利用可能なSkills

### skill-developer（Meta-Skill）
**目的：** Claude Code skillsの作成と管理

**ファイル：** リソースファイル7個（合計426行）

**使用時期：**
- 新しいskills作成
- Skill構造の理解
- skill-rules.json作業
- Skillアクティベーションのデバッグ

**カスタマイズ：** ✅ 不要 - そのままコピー

**[Skillを見る →](skill-developer/)**

---

### backend-dev-guidelines
**目的：** Node.js/Express/TypeScript開発パターン

**ファイル：** リソースファイル12個（メイン304行 + リソース）

**カバー内容：**
- Layered architecture（Routes → Controllers → Services → Repositories）
- BaseControllerパターン
- Prisma databaseアクセス
- Sentry error tracking
- Zod検証
- UnifiedConfigパターン
- Dependency injection
- テスト戦略

**使用時期：**
- API routes作成/修正
- Controllersまたはservices構築
- Prismaを使用したdatabase作業
- Error tracking設定

**カスタマイズ：** ⚠️ skill-rules.jsonの`pathPatterns`をbackendディレクトリに合わせて更新

**pathPatterns例：**
```json
{
  "pathPatterns": [
    "src/api/**/*.ts",       // src/apiを持つ単一アプリ
    "backend/**/*.ts",       // Backendディレクトリ
    "services/*/src/**/*.ts" // Multi-service monorepo
  ]
}
```

**[Skillを見る →](backend-dev-guidelines/)**

---

### frontend-dev-guidelines
**目的：** React/TypeScript/MUI v7開発パターン

**ファイル：** リソースファイル11個（メイン398行 + リソース）

**カバー内容：**
- 最新Reactパターン（Suspense、lazy loading）
- データfetchingのためのuseSuspenseQuery
- MUI v7 styling（`size={{}}`propを持つGrid）
- TanStack Router
- ファイル構成（features/パターン）
- パフォーマンス最適化
- TypeScriptベストプラクティス

**使用時期：**
- Reactコンポーネント作成
- TanStack Queryでデータfetching
- MUI v7でstyling
- Routing設定

**カスタマイズ：** ⚠️ `pathPatterns`更新 + React/MUI使用確認

**pathPatterns例：**
```json
{
  "pathPatterns": [
    "src/**/*.tsx",          // 単一Reactアプリ
    "frontend/src/**/*.tsx", // Frontendディレクトリ
    "apps/web/**/*.tsx"      // Monorepo webアプリ
  ]
}
```

**注意：** このskillはMUI v6→v7非互換性を防ぐために**guardrail**（enforcement: "block"）として設定されています。

**[Skillを見る →](frontend-dev-guidelines/)**

---

### route-tester
**目的：** JWT cookie authを使用した認証済みAPI routesテスト

**ファイル：** メインファイル1個（389行）

**カバー内容：**
- JWT cookie基盤認証テスト
- test-auth-route.js scriptパターン
- Cookie認証を使用したcURL
- Auth問題デバッグ
- POST/PUT/DELETE操作テスト

**使用時期：**
- API endpointsテスト
- 認証デバッグ
- Route機能検証

**カスタマイズ：** ⚠️ JWT cookie auth設定が必要

**先に確認：** 「JWT cookie基盤認証を使用していますか？」
- YESの場合：コピーしてservice URLsをカスタマイズ
- NOの場合：スキップまたはauth方法に合わせて調整

**[Skillを見る →](route-tester/)**

---

### error-tracking
**目的：** Sentry error trackingとモニタリングパターン

**ファイル：** メインファイル1個（~250行）

**カバー内容：**
- Sentry v8初期化
- Error captureパターン
- Breadcrumbsとuser context
- Performanceモニタリング
- ExpressとReactとの統合

**使用時期：**
- Error tracking設定
- Exceptionキャプチャ
- Error context追加
- 本番問題デバッグ

**カスタマイズ：** ⚠️ backendに合わせて`pathPatterns`更新

**[Skillを見る →](error-tracking/)**

---

## プロジェクトにSkillを追加する方法

### クイック統合

**Claude Codeの場合：**
```
ユーザー：「私のプロジェクトにbackend-dev-guidelines skillを追加して」

Claudeは以下をすべき：
1. プロジェクト構造について質問
2. Skillディレクトリをコピー
3. ユーザーのパスでskill-rules.jsonを更新
4. 統合を検証
```

完全な手順は[CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を参照。

### 手動統合

**Step 1: Skillディレクトリをコピー**
```bash
cp -r claude-code-infrastructure-showcase/.claude/skills/backend-dev-guidelines \\
      your-project/.claude/skills/
```

**Step 2: skill-rules.jsonを更新**

ない場合は作成：
```bash
cp claude-code-infrastructure-showcase/.claude/skills/skill-rules.json \\
   your-project/.claude/skills/
```

その後プロジェクトに合わせて`pathPatterns`をカスタマイズ：
```json
{
  "skills": {
    "backend-dev-guidelines": {
      "fileTriggers": {
        "pathPatterns": [
          "YOUR_BACKEND_PATH/**/*.ts"  // ← これを更新！
        ]
      }
    }
  }
}
```

**Step 3: テスト**
- Backendディレクトリでファイル編集
- Skillが自動的にアクティベートされるべき

---

## skill-rules.json設定

### 概要と目的

`skill-rules.json`は複数のskillsのアクティベーション条件、トリガー、権限制御を中央で管理する核心設定ファイルです。プロジェクト/組織レベルで一貫した開発ガイドラインと安全装置を実装します。

**Skillsアクティベーション基準：**
- ユーザープロンプトの**keywords**（「backend」、「API」、「route」）
- **intentPatterns**（正規表現でユーザー意図マッチング）
- **pathPatterns**（ファイルパス基盤 - backendファイル編集）
- **contentPatterns**（ファイル内容基盤 - コードにPrismaクエリ含む）

---

### 全体構造

```json
{
  "version": "1.0",
  "description": "Skill activation triggers for Claude Code",
  "skills": {
    "skill-name": {
      "type": "domain" | "guardrail",
      "enforcement": "suggest" | "block" | "warn",
      "priority": "critical" | "high" | "medium" | "low",
      "description": "Skill説明",

      "promptTriggers": {
        "keywords": ["キーワード", "リスト"],
        "intentPatterns": ["(create|add).*?(route|API)"]
      },

      "fileTriggers": {
        "pathPatterns": ["backend/**/*.ts"],
        "pathExclusions": ["**/*.test.ts"],
        "contentPatterns": ["import.*Prisma"]
      },

      "blockMessage": "ブロック時に表示するメッセージ（enforcement: block時）",

      "skipConditions": {
        "sessionSkillUsed": true,
        "fileMarkers": ["@skip-validation"],
        "envOverride": "SKIP_SKILL_NAME"
      }
    }
  }
}
```

---

### Enforcementタイプ詳細

| Type | 動作方式 | 使用時期 | 例 |
|------|----------|----------|-----|
| **suggest** | 提案のみでブロックしない | 一般ガイドライン、ベストプラクティス | backend-dev-guidelines |
| **warn** | 警告メッセージ出力後進行許可 | 注意が必要な作業 | 危険な作業警告 |
| **block** | 条件満たした時実行ブロック（guardrail）| 必須遵守事項、セキュリティ | frontend-dev-guidelines（MUI v7強制）|

**"block"使用推奨：**
- Breaking changes防止（MUI v6→v7非互換性）
- 重要なdatabase操作
- セキュリティに敏感なコード
- 必須検証が必要な場合

**"suggest"使用推奨：**
- 一般ベストプラクティス案内
- ドメイン別ガイド
- コード構成提案
- オプションの最適化

**"warn"使用推奨：**
- 危険だが許容可能な作業
- パフォーマンス影響がある作業
- Deprecated機能使用警告

---

### Priority Levels

複数のskillsが同時にマッチした時の優先順位を決定：

| Priority | 意味 | 使用時期 |
|----------|------|----------|
| **critical** | 常にトリガー | セキュリティ/必須ルール |
| **high** | ほとんどのマッチでトリガー | 重要なガイドライン |
| **medium** | 明確なマッチでトリガー | 一般パターン |
| **low** | 明示的マッチでのみトリガー | オプション提案 |

**例：**
- セキュリティルール（block）：`priority: "critical"`
- Frontendガイドライン：`priority: "high"`
- コードスタイル提案：`priority: "low"`

---

### Triggers詳細説明

#### promptTriggers
ユーザーのプロンプト（入力）を分析してskillアクティベーション：

```json
"promptTriggers": {
  "keywords": [
    "backend",
    "controller",
    "service",
    "API"
  ],
  "intentPatterns": [
    "(create|add|implement).*?(route|endpoint|API)",
    "(how to|best practice).*?(backend|controller)"
  ]
}
```

- **keywords**: シンプルなキーワードマッチング（大文字小文字無視）
- **intentPatterns**: 正規表現でユーザー意図把握（より正確なマッチング）

#### fileTriggers
ファイルパスや内容に基づいてskillアクティベーション：

```json
"fileTriggers": {
  "pathPatterns": [
    "backend/**/*.ts",
    "src/api/**/*.ts"
  ],
  "pathExclusions": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ],
  "contentPatterns": [
    "router\\.",
    "export.*Controller",
    "prisma\\."
  ]
}
```

**pathPatterns vs contentPatterns：**

| タイプ | 検査対象 | 例 | 用途 |
|--------|----------|-----|------|
| **pathPatterns** | ファイルパス/名 | `"backend/**/*.ts"` | 特定ディレクトリ/ファイルタイプ |
| **contentPatterns** | ファイル内容 | `"import.*Prisma"` | 特定コードパターン検出 |
| **pathExclusions** | 除外するパス | `"**/*.test.ts"` | テストファイル除外 |

---

### skipConditions（例外処理）

特定条件でルールをスキップするメカニズム：

```json
"skipConditions": {
  "sessionSkillUsed": true,
  "fileMarkers": ["@skip-validation", "// @ignore-skill"],
  "envOverride": "SKIP_FRONTEND_GUIDELINES"
}
```

**各条件説明：**
- **sessionSkillUsed**: セッションで既にskillを使用した場合スキップ
- **fileMarkers**: ファイルに特定コメントがあればスキップ（開発者が意図的に回避）
- **envOverride**: 環境変数設定時スキップ

**使用例：**
```typescript
// @skip-validation
// このファイルはレガシーコードでfrontend-dev-guidelinesに従わない
import { makeStyles } from '@material-ui/core';
```

---

### ベストプラクティス

#### 1. 明確なTrigger定義
```json
// ❌ 悪い: 広すぎる
"keywords": ["backend"]

// ✅ 良い: 具体的
"keywords": ["backend", "controller", "service", "repository"],
"intentPatterns": ["(create|add|implement).*?(route|controller)"]
```

#### 2. pathPatternsをプロジェクトに合わせる
```json
// ❌ 悪い: 例のパスそのまま
"pathPatterns": ["blog-api/src/**/*.ts"]

// ✅ 良い: 実際のプロジェクトパス
"pathPatterns": [
  "src/api/**/*.ts",
  "backend/**/*.ts"
]
```

#### 3. 適切なEnforcement選択
```json
// 一般ガイドライン
"enforcement": "suggest"

// 必須遵守事項（セキュリティ、互換性）
"enforcement": "block"

// 警告のみ表示
"enforcement": "warn"
```

#### 4. skipConditions活用
繰り返しの警告を防止し例外状況を明確に処理：
```json
"skipConditions": {
  "sessionSkillUsed": true,  // セッションごと1回のみ警告
  "fileMarkers": ["@legacy"]  // レガシーコード例外
}
```

---

### カスタマイズガイド

プロジェクトに合わせて調整すべき項目：

1. **pathPatterns**: 実際のプロジェクトディレクトリ構造に合わせて変更
   ```json
   "pathPatterns": ["YOUR_PROJECT_PATH/**/*.ts"]
   ```

2. **keywords**: ドメイン別用語追加
   ```json
   "keywords": ["your-domain-term", "project-specific"]
   ```

3. **intentPatterns**: チームの作業パターンに合った正規表現
   ```json
   "intentPatterns": ["team-specific-pattern"]
   ```

---

### 実際の使用例

**Domain Skill（suggest）：**
```json
"backend-dev-guidelines": {
  "type": "domain",
  "enforcement": "suggest",
  "priority": "high",
  "promptTriggers": {
    "keywords": ["backend", "API", "controller"],
    "intentPatterns": ["(create|add).*?(route|controller)"]
  },
  "fileTriggers": {
    "pathPatterns": ["backend/**/*.ts"],
    "contentPatterns": ["export.*Controller"]
  }
}
```

**Guardrail Skill（block）：**
```json
"frontend-dev-guidelines": {
  "type": "guardrail",
  "enforcement": "block",
  "priority": "high",
  "blockMessage": "⚠️ BLOCKED - MUI v7パターン必要",
  "fileTriggers": {
    "pathPatterns": ["frontend/**/*.tsx"],
    "contentPatterns": ["makeStyles", "@material-ui/core"]
  },
  "skipConditions": {
    "sessionSkillUsed": true,
    "fileMarkers": ["@skip-validation"]
  }
}
```

---

## 自分だけのSkillsを作る

以下の完全なガイドは**skill-developer** skillを参照：
- Skill YAML frontmatter構造
- Resourceファイル構成
- Triggerパターン設計
- Skillアクティベーションテスト

**クイックテンプレート：**
```markdown
---
name: my-skill
description: このskillがすること
---

# My Skillタイトル

## 目的
[このskillが存在する理由]

## Skill使用時期
[自動アクティベーションシナリオ]

## クイックリファレンス
[主要パターンと例]

## Resourceファイル
- [topic-1.md](resources/topic-1.md)
- [topic-2.md](resources/topic-2.md)
```

---

## トラブルシューティング

### Skillがアクティベートされない

**確認事項：**
1. `.claude/skills/`にskillディレクトリがありますか？
2. `skill-rules.json`にskillがリストされていますか？
3. `pathPatterns`がファイルとマッチしていますか？
4. Hooksがインストールされて動作していますか？
5. settings.jsonが正しく設定されていますか？

**デバッグ：**
```bash
# Skill存在確認
ls -la .claude/skills/

# skill-rules.json検証
cat .claude/skills/skill-rules.json | jq .

# Hooksが実行可能か確認
ls -la .claude/hooks/*.sh

# Hookを手動でテスト
./.claude/hooks/skill-activation-prompt.sh
```

### Skillが頻繁にアクティベートされすぎる

skill-rules.jsonを更新：
- キーワードをより具体的に
- `pathPatterns`範囲を狭める
- `intentPatterns`の具体性を高める

### Skillが全くアクティベートされない

skill-rules.jsonを更新：
- より多くのキーワードを追加
- `pathPatterns`範囲を広げる
- より多くの`intentPatterns`を追加

---

## Claude Code向け

**ユーザーのためにskillを統合する時：**

1. **まず[CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を読む**
2. プロジェクト構造について質問
3. skill-rules.jsonの`pathPatterns`をカスタマイズ
4. Skillファイルにハードコードされたパスがないか確認
5. 統合後にアクティベーションテスト

**一般的な間違い：**
- 例のパスを維持（blog-api/、frontend/）
- Monorepo vs 単一アプリについて聞かない
- カスタマイズなしでskill-rules.jsonをコピー

---

## 次のステップ

1. **シンプルに始める：** タスクにマッチするskillを1つ追加
2. **アクティベーション検証：** 関連ファイル編集、skillが提案されるべき
3. **さらに追加：** 最初のskillが動作したら他のものを追加
4. **カスタマイズ：** ワークフローに合わせてtriggersを調整

**質問がありますか？** 包括的な統合手順は[CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を参照してください。
