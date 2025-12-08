# Claude Integration Guide

**Claude Code向けガイド：** ユーザーがこのshowcase repositoryのコンポーネントを自分のプロジェクトに統合するよう依頼する際、以下の指示に注意深く従ってください。

---

## 概要

このrepositoryはClaude Code infrastructureコンポーネントの**リファレンスライブラリ**です。ユーザーは特定の部分を自分のプロジェクトに統合するのを手伝ってほしいと依頼します。あなたの役割は：

1. **プロジェクト構造について明確な質問をする**
2. **適切なファイルをコピーする**
3. **設定をカスタマイズする**
4. **統合が正しく動作するか検証する**

**核心原則：** プロジェクト構造を仮定する前に必ず質問してください。あるプロジェクトで動作するものが別のプロジェクトでは動作しない可能性があります。

---

## Tech Stack互換性確認

**重要：** Skillを統合する前に、ユーザーのtech stackがskill要件と一致するか確認してください。

### Frontend Skills

**frontend-dev-guidelines要件：**
- React (18+)
- MUI v7
- TanStack Query
- TanStack Router
- TypeScript

**統合前に質問：**
「ReactとMUI v7を使用していますか？」

**いいえの場合：**
```
frontend-dev-guidelines skillはReact + MUI v7専用に設計されています。以下が可能です：
1. これをテンプレートとして使用して[ユーザーのstack]に合わせた類似skillを作成
2. Framework非依存のパターンを抽出（ファイル構成、パフォーマンスなど）
3. 関係ない場合はこのskillをスキップ

どれを希望しますか？
```

### Backend Skills

**backend-dev-guidelines要件：**
- Node.js/Express
- TypeScript
- Prisma ORM
- Sentry

**統合前に質問：**
「Node.jsとExpress、Prismaを使用していますか？」

**いいえの場合：**
```
backend-dev-guidelines skillはExpress/Prisma用に設計されています。以下が可能です：
1. これをテンプレートとして使用して[ユーザーのstack]に合わせた類似ガイドラインを作成
2. アーキテクチャパターンを抽出（layered architectureはすべてのframeworkに適用可能）
3. このskillをスキップ

どれを希望しますか？
```

### Tech-AgnosticなSkills

すべてのtech stackで動作：
- ✅ **skill-developer** - Meta-skill、技術要件なし
- ✅ **route-tester** - JWT cookie認証のみ必要（framework非依存）
- ✅ **error-tracking** - Sentryはほとんどのstackと互換

---

## 一般統合パターン

ユーザーが**「[コンポーネント]を私のプロジェクトに追加して」**と言う時：

1. コンポーネントタイプを識別（skill/hook/agent/command）
2. **TECH STACK互換性を確認**（frontend/backend skillsの場合）
3. プロジェクト構造について質問
4. ファイルをコピーまたはユーザーのstackに合わせて調整
5. 設定をカスタマイズ
6. 統合を検証
7. 次のステップを提供

---

## Skills統合

### ステップバイステッププロセス

**ユーザーがskillを要求する時**（例：「backend-dev-guidelinesを追加して」）：

#### 1. プロジェクトを理解する

**以下の質問をしてください：**
- 「プロジェクト構造はどうなっていますか？単一アプリ、monorepo、またはmulti-service？」
- 「[backend/frontend]コードはどこにありますか？」
- 「どのframeworks/technologiesを使用していますか？」

#### 2. Skillをコピー

```bash
cp -r /path/to/showcase/.claude/skills/[skill-name] \\
      $CLAUDE_PROJECT_DIR/.claude/skills/
```

#### 3. skill-rules.jsonを処理

**存在確認：**
```bash
ls $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**ない場合（存在しない）：**
- showcaseからテンプレートをコピー
- ユーザーが望まないskillsを削除
- プロジェクトに合わせてカスタマイズ

**ある場合（存在する）：**
- 現在のskill-rules.jsonを読む
- 新しいskillエントリを追加
- 既存のskillsが壊れないよう慎重にマージ

#### 4. Pathパターンをカスタマイズ

**重要：** skill-rules.jsonの`pathPatterns`をユーザーの構造に合わせて更新：

**例 - ユーザーがmonorepoを持つ場合：**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "packages/api/src/**/*.ts",
        "packages/server/src/**/*.ts",
        "apps/backend/**/*.ts"
      ]
    }
  }
}
```

**例 - ユーザーが単一backendを持つ場合：**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "src/**/*.ts",
        "backend/**/*.ts"
      ]
    }
  }
}
```

**安全な汎用パターン**（確信がない時）：
```json
{
  "pathPatterns": [
    "**/*.ts",          // すべてのTypeScriptファイル
    "src/**/*.ts",      // 一般的なsrcディレクトリ
    "backend/**/*.ts"   // 一般的なbackendディレクトリ
  ]
}
```

#### 5. 統合を検証

```bash
# Skillがコピーされたか確認
ls -la $CLAUDE_PROJECT_DIR/.claude/skills/[skill-name]

# skill-rules.json構文を検証
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
```

**ユーザーに伝える：** 「[their-backend-path]でファイルを編集するとskillがアクティベートされます。」

---

### Skill別注意事項

#### backend-dev-guidelines
- **技術要件：** Node.js/Express、Prisma、TypeScript、Sentry
- **質問：** 「ExpressとPrismaを使用していますか？」「backendコードはどこにありますか？」
- **異なるstackの場合：** これをテンプレートとして調整すると提案
- **カスタマイズ：** pathPatterns
- **例パス：** `api/`、`server/`、`backend/`、`services/*/src/`
- **調整Tips：** アーキテクチャパターン（Routes→Controllers→Services）はほとんどのframeworksに適用

#### frontend-dev-guidelines
- **技術要件：** React 18+、MUI v7、TanStack Query/Router、TypeScript
- **質問：** 「ReactとMUI v7を使用していますか？」「frontendコードはどこにありますか？」
- **異なるstackの場合：** 調整バージョン作成を提案（Vue、Angularなど）
- **カスタマイズ：** pathPatterns + すべてのframework別例
- **例パス：** `frontend/`、`client/`、`web/`、`apps/web/src/`
- **調整Tips：** ファイル構成とパフォーマンスパターンは適用されるが、コンポーネントコードは適用されない

#### route-tester
- **技術要件：** JWT cookie基盤認証（framework非依存）
- **質問：** 「JWT cookie基盤認証を使用していますか？」
- **いいえの場合：** 「このskillはJWT cookies用に設計されています。[ユーザーのauthタイプ]に合わせて調整またはスキップしますか？」
- **カスタマイズ：** Service URLs、認証パターン
- **互換：** JWT cookiesを使用するすべてのbackend framework

#### error-tracking
- **技術要件：** Sentry（ほとんどのbackendsと互換）
- **質問：** 「Sentryを使用していますか？」「backendコードはどこにありますか？」
- **Sentryを使用しない場合：** 「[ユーザーのerror tracking]のテンプレートとして使用しますか？」
- **カスタマイズ：** pathPatterns
- **調整Tips：** Error tracking哲学は他のツールに適用可能（Rollbar、Bugsnagなど）

#### skill-developer
- **技術要件：** なし！
- **そのままコピー** - meta-skill、完全に汎用、すべてのtech stackのskill作成を教える

---

## 異なるTech StackにSkillsを調整

ユーザーのtech stackがskill要件と異なる場合、以下のオプションがあります：

### オプション1：既存Skillを調整（推奨）

**使用時：** ユーザーが異なる技術のための類似ガイドラインを望む時

**プロセス：**
1. **開始点としてskillをコピー：**
   ```bash
   cp -r showcase/.claude/skills/frontend-dev-guidelines \\
         $CLAUDE_PROJECT_DIR/.claude/skills/vue-dev-guidelines
   ```

2. **変更が必要なものを識別：**
   - Framework別コード例（React → Vue）
   - Library APIs（MUI → Vuetify/PrimeVue）
   - Import文
   - コンポーネントパターン

3. **適用されるものを維持：**
   - ファイル構成原則
   - パフォーマンス最適化戦略
   - TypeScript標準
   - 一般的なベストプラクティス

4. **体系的に例を置換：**
   - ユーザーのstackで同等のパターンを質問
   - コード例をユーザーのframeworkに合わせて更新
   - 全体構造とセクションを維持

5. **Skill名とtriggersを更新：**
   - Skill名を適切に変更
   - skill-rules.json triggersをユーザーのstackに合わせて更新
   - アクティベーションをテスト

**例 - Vueにfrontend-dev-guidelinesを調整：**
```
React skill構造を基にvue-dev-guidelinesを作成します：
- React.FC → Vue defineComponentに置換
- useSuspenseQuery → Vue composablesに置換
- MUI components → [ユーザーのcomponent library]に置換
- 維持：ファイル構成、パフォーマンスパターン、TypeScriptガイドライン

数分かかります。よろしいですか？
```

### オプション2：Framework非依存パターンを抽出

**使用時：** Stackが非常に異なるがコア原則は適用可能な時

**プロセス：**
1. 既存skillを読む
2. Framework非依存パターンを識別：
   - Layered architecture（backend）
   - ファイル構成戦略
   - パフォーマンス最適化原則
   - テスト戦略
   - Error処理哲学

3. そのパターンだけで新skillを作成
4. ユーザーが後でframework別例を追加可能

**例：**
```
backend-dev-guidelinesはExpressを使用していますが、layered architecture
（Routes → Controllers → Services → Repositories）はDjangoでも動作します。

以下の内容でskillを作成できます：
- Layered architectureパターン
- 関心の分離原則
- Error処理ベストプラクティス
- テスト戦略

その後、パターンを確立しながらDjango別例を追加できます。
```

### オプション3：リファレンスとしてのみ使用

**使用時：** 調整するには異なりすぎるが、ユーザーがインスピレーションを望む時

**プロセス：**
1. ユーザーが既存skillを閲覧
2. ゼロから新skillを作成する手助け
3. 既存skillの構造をテンプレートとして使用
4. モジュラーパターンに従う（main + resourceファイル）

### Tech Stack間で一般的に適用されるもの

**アーキテクチャ＆構成：**
- ✅ Layered architecture（Routes/Controllers/Servicesパターン）
- ✅ 関心の分離
- ✅ ファイル構成戦略（features/パターン）
- ✅ Progressive disclosure（main + resourceファイル）
- ✅ データアクセスのためのRepositoryパターン

**開発プラクティス：**
- ✅ Error処理哲学
- ✅ Input検証の重要性
- ✅ テスト戦略
- ✅ パフォーマンス最適化原則
- ✅ TypeScriptベストプラクティス

**Framework別コード：**
- ❌ React hooks → Vue/Angularに適用されない
- ❌ MUI components → 他のcomponent libraries
- ❌ Prisma queries → 他のORM構文
- ❌ Express middleware → 他のframeworkパターン
- ❌ Routing実装 → Framework別

### 調整 vs スキップ推奨時期

**調整を推奨：**
- ユーザーが自分のstackのための類似ガイドラインを望む時
- コアパターンが適用される時（layered architectureなど）
- ユーザーがframework別例を手伝う時間がある時

**スキップを推奨：**
- Stackが完全に異なる時
- ユーザーがそのパターンを必要としない時
- 調整に時間がかかりすぎる時
- ユーザーがゼロから作成を好む時

---

## Hooks統合

### 必須Hooks（常に安全にコピー可能）

#### skill-activation-prompt（UserPromptSubmit）

**目的：** ユーザープロンプトに基づいてskillsを自動提案

**統合（カスタマイズ不要）：**

```bash
# 両ファイルをコピー
cp showcase/.claude/hooks/skill-activation-prompt.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/
cp showcase/.claude/hooks/skill-activation-prompt.ts \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 実行可能にする
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# 必要に応じてdependenciesをインストール
if [ -f "showcase/.claude/hooks/package.json" ]; then
  cp showcase/.claude/hooks/package.json \\
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && npm install
fi
```

**settings.jsonに追加：**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**このhookは完全に汎用的です** - どこでも動作、カスタマイズ不要！

#### post-tool-use-tracker（PostToolUse）

**目的：** Context管理のためのファイル変更追跡

**統合（カスタマイズ不要）：**

```bash
# ファイルをコピー
cp showcase/.claude/hooks/post-tool-use-tracker.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 実行可能にする
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
```

**settings.jsonに追加：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**このhookは完全に汎用的です** - プロジェクト構造を自動検出！

---

### オプションHooks（多くのカスタマイズが必要）

#### tsc-check.shとtrigger-build-resolver.sh（Stop hooks）

⚠️ **警告：** これらのhooksは特定のmulti-service monorepo構造用に設定されています。

**統合前に質問：**
1. 「複数のTypeScript servicesがあるmonorepoですか？」
2. 「serviceディレクトリ名は何ですか？」
3. 「tsconfig.jsonファイルはどこにありますか？」

**シンプルなプロジェクトの場合（単一service）：**
- **これらのhooksをスキップ推奨**
- 単一serviceプロジェクトには過剰
- ユーザーが代わりに`tsc --noEmit`を手動で実行可能

**複雑なプロジェクトの場合（multi-service monorepo）：**

1. ファイルをコピー
2. **必ず編集** tsc-check.sh - このセクションを探す：
```bash
case "$repo" in
    email|exports|form|frontend|projects|uploads|users|utilities|events|database)
        echo "$repo"
        return 0
        ;;
esac
```

3. ユーザーの実際のservice名で置換：
```bash
case "$repo" in
    api|web|auth|payments|notifications)  # ← ユーザーのservices
        echo "$repo"
        return 0
        ;;
esac
```

4. settings.jsonに追加する前に手動でテスト：
```bash
./.claude/hooks/tsc-check.sh
```

**重要：** このhookが失敗するとStopイベントをブロックします。設定で動作すると確信できる時だけ追加してください。

---

## Agents統合

**Agentsはスタンドアロンです** - 統合が最も簡単！

### 標準Agent統合

```bash
# Agentファイルをコピー
cp showcase/.claude/agents/[agent-name].md \\
   $CLAUDE_PROJECT_DIR/.claude/agents/
```

**完了！** Agentsは即座に動作し、設定不要です。

### ハードコードされたパスを確認

一部のagentsはパスを参照する場合があります。**コピー前にagentファイルを読み、以下を確認：**

- `~/git/old-project/` → `$CLAUDE_PROJECT_DIR`または`.`に変更
- `/root/git/project/` → `$CLAUDE_PROJECT_DIR`または`.`に変更
- ハードコードされたスクリーンショットパス → ユーザーにスクリーンショットをどこに保存するか質問

**発見された場合は更新：**
```bash
sed -i 's|~/git/old-project/|.|g' $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
sed -i 's|/root/git/.*PROJECT.*DIR|$CLAUDE_PROJECT_DIR|g' \\
    $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
```

### Agent別注意事項

**auth-route-tester / auth-route-debugger：**
- ユーザープロジェクトにJWT cookie基盤認証が必要
- 質問：「AuthにJWT cookiesを使用していますか？」
- いいえの場合：「これらのagentsはJWT cookie auth用です。スキップまたは調整しますか？」

**frontend-error-fixer：**
- スクリーンショットパスを参照する可能性
- 質問：「スクリーンショットをどこに保存しますか？」

**その他すべてのagents：**
- そのままコピー、完全に汎用

---

## Slash Commands統合

```bash
# Commandファイルをコピー
cp showcase/.claude/commands/[command].md \\
   $CLAUDE_PROJECT_DIR/.claude/commands/
```

### パスのカスタマイズ

Commandsがdev docsパスを参照する場合があります。**確認して更新：**

**dev-docsとdev-docs-update：**
- `dev/active/`パス参照を探す
- 質問：「dev documentationをどこに保存しますか？」
- Commandファイルのパスを更新

**route-research-for-testing：**
- Serviceパスを参照する可能性
- API構造について質問

---

## 一般的なパターン＆ベストプラクティス

### パターン：プロジェクト構造について質問

**仮定しない：**
- ❌ 「blog-api serviceにこれを追加します」
- ❌ 「frontendディレクトリに合わせて設定します」

**質問する：**
- ✅ 「プロジェクト構造はどうなっていますか？Monorepoまたは単一アプリ？」
- ✅ 「backendコードはどこにありますか？」
- ✅ 「workspacesを使用していますか、または複数のservicesがありますか？」

### パターン：skill-rules.jsonカスタマイズ

**ユーザーがworkspacesありのmonorepoを持つ場合：**
```json
{
  "pathPatterns": [
    "packages/*/src/**/*.ts",
    "apps/*/src/**/*.tsx"
  ]
}
```

**ユーザーがNx monorepoを持つ場合：**
```json
{
  "pathPatterns": [
    "apps/api/src/**/*.ts",
    "libs/*/src/**/*.ts"
  ]
}
```

**ユーザーがシンプルな構造を持つ場合：**
```json
{
  "pathPatterns": [
    "src/**/*.ts",
    "backend/**/*.ts"
  ]
}
```

### パターン：settings.json統合

**showcaseのsettings.jsonを直接コピーしない！**

代わりに必要なセクションを**抽出してマージ**：

1. 既存settings.jsonを読む
2. 望むhook設定を追加
3. 既存設定を保持

**マージ例：**
```json
{
  // ... 既存設定 ...
  "hooks": {
    // ... 既存hooks ...
    "UserPromptSubmit": [  // ← このセクションを追加
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 検証チェックリスト

統合後に**以下を検証：**

```bash
# 1. Hooksが実行可能か
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
# 次のように表示されるべき：-rwxr-xr-x

# 2. skill-rules.jsonが有効なJSONか
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
# エラーなしでパースされるべき

# 3. Hook dependenciesがインストールされている（TypeScript hooksの場合）
ls $CLAUDE_PROJECT_DIR/.claude/hooks/node_modules/
# package.jsonがあればパッケージが表示されるべき

# 4. Settings.jsonが有効なJSONか
cat $CLAUDE_PROJECT_DIR/.claude/settings.json | jq .
# エラーなしでパースされるべき
```

**その後ユーザーにテストを依頼：**
- 「[relevant-path]でファイルを編集してみてください - skillがアクティベートされるはずです」
- 「[topic]について質問してみてください - skillを提案するはずです」

---

## 避けるべき一般的な間違い

### ❌ しないで：settings.jsonをそのままコピー
**理由：** Stop hooksが存在しないservicesを参照
**代わりに：** UserPromptSubmitとPostToolUseセクションだけを抽出

### ❌ しないで：例のservice名を維持
**理由：** ユーザーにはblog-api、auth-serviceなどがない
**代わりに：** 実際の構造について質問して更新

### ❌ しないで：Hooksを実行可能にするのをスキップ
**理由：** 実行権限なしではhooksが実行されない
**代わりに：** コピー後は常に`chmod +x`を実行

### ❌ しないで：Monorepo構造を仮定
**理由：** ほとんどのプロジェクトは単一service
**代わりに：** 先に質問してからカスタマイズ

### ❌ しないで：すべてのskillsを一度に追加
**理由：** 圧倒的で、すべてが関連するわけではない
**代わりに：** 実際に必要なものは何か質問

### ❌ しないで：テストなしでStop hooksをコピー
**理由：** 失敗するとStopイベントをブロックする可能性
**代わりに：** まず手動でテスト、動作する時だけ追加

---

## 例統合会話

### 例1：Backend Skill追加

**ユーザー：** 「backend-dev-guidelines skillを私のプロジェクトに追加して」

**Claudeは以下のように応答すべき：**
```
backend-dev-guidelines skillを追加します。まずプロジェクト構造を
理解する必要があります：

1. 複数servicesがあるmonorepoですか、それとも単一backendアプリですか？
2. backendコードはどこにありますか？（例：src/、backend/、server/）
3. Express.jsを使用していますか？
```

**ユーザー：** 「単一アプリで、コードはsrc/api/にあるよ。Express使ってる」

**Claudeは以下をすべき：**
1. Skillディレクトリをコピー
2. skill-rules.jsonをpathPattern: `"src/api/**/*.ts"`で更新
3. 正しくコピーされたか検証
4. 応答：
```
✅ backend-dev-guidelines skill追加完了！

設定：
- src/api/でファイル編集時にskillがトリガー
- "controller"、"service"、"route"、"API"などのキーワードでアクティベート

テスト：src/api/でファイルを編集するとskill提案が表示されます。
```

### 例2：Hooks設定

**ユーザー：** 「skillアクティベーションhooksを設定して」

**Claudeは以下のように応答すべき：**
```
skill自動アクティベーションシステムを設定します。2つのhooksが必要です：
1. skill-activation-prompt - プロンプトに基づいてskillsを提案
2. post-tool-use-tracker - ファイル変更を追跡

.claude/settings.jsonファイルは既にありますか、それとも新しく作成しますか？
```

**ユーザー：** 「既にsettings.jsonがある」

**Claudeは以下をすべき：**
1. settings.jsonを読む
2. 両hookファイルをコピー
3. hooksディレクトリでnpm installを実行
4. settings.jsonにhook設定を追加（既存設定を保持）
5. Hooksを実行可能にする
6. 応答：
```
✅ Skillアクティベーションhooksインストール完了！

実行した作業：
- skill-activation-prompt hookをコピー
- post-tool-use-tracker hookをコピー
- Dependenciesをインストール
- settings.jsonに追加

テスト：「backend開発を手伝って」と質問するとbackend-dev-guidelines skillを
提案します。
```

### 例3：Agent追加

**ユーザー：** 「code-architecture-reviewer agentを追加して」

**Claudeは以下のように応答すべき：**
```
code-architecture-reviewer agentはアーキテクチャ問題とベストプラクティスに
ついてコードをレビューします。このagentはスタンドアロンで設定不要です。

.claude/agents/ディレクトリにコピーします。
```

**その後：**
1. Agentファイルをコピー
2. 応答：
```
✅ code-architecture-reviewer agent追加完了！

Task toolを実行して使用：
- subagent_type: "code-architecture-reviewer"
- prompt: レビューするコードの説明

Agentがコードをレビューしてアーキテクチャフィードバックを提供します。
```

### 例4：Tech Stack不一致

**ユーザー：** 「Vueプロジェクトにfrontend-dev-guidelines skillを追加して」

**Claudeは以下のように応答すべき：**
```
Vueを使用しているのですね。frontend-dev-guidelines skillはReact + MUI v7
専用に設計されています。いくつかのオプションがあります：

オプション1（推奨）：React skillをテンプレートとしてvue-dev-guidelines skillを
作成できます。以下を調整します：
- Vue用コンポーネントパターン（Composition API、defineComponent）
- MUI例をあなたのcomponent libraryに置換
- Vueパターンに合わせてdata fetchingを調整
- 維持：ファイル構成、パフォーマンス原則、TypeScript標準

オプション2：React別コードなしでframework非依存パターンだけを抽出（ファイル
構成、パフォーマンス、テスト）

オプション3：スキップしてゼロからVueガイドラインを作成

どれを希望しますか？
```

**ユーザー：** 「オプション1がいいかな、Vuetify使ってる」

**Claudeは以下をすべき：**
1. frontend-dev-guidelinesをvue-dev-guidelinesとしてコピー
2. 各resourceファイルを処理
3. ReactパターンをVue Composition APIに置換
4. MUIコンポーネントをVuetifyに置換
5. skill-rules.jsonをVue別triggers（.vueファイル）で更新
6. 応答：
```
✅ Vue 3 + Vuetifyに合わせて調整したvue-dev-guidelines skill作成完了！

変更内容：
- React.FCコンポーネント → Vue defineComponent
- MUI v7例 → Vuetifyコンポーネント
- TanStack Query → あなたのdata fetchingアプローチ
- ファイルパターン → .vueファイル

維持内容：
- ファイル構成（features/パターン）
- パフォーマンス最適化戦略
- TypeScript標準
- Loading/error state原則

.vueファイルを編集してみてください - skillがアクティベートされます。
```

---

## クイックリファレンステーブル

### カスタマイズが必要なものは？

| コンポーネント | 技術要件 | カスタマイズ | 質問内容 |
|---------------|----------|--------------|----------|
| **skill-developer** | なし | ✅ 不要 | そのままコピー |
| **backend-dev-guidelines** | Express/Prisma/Node | ⚠️ パス + 技術確認 | 「Express/Prisma使用？」「backend位置？」 |
| **frontend-dev-guidelines** | React/MUI v7 | ⚠️⚠️ パス + framework | 「React/MUI v7使用？」「frontend位置？」 |
| **route-tester** | JWT cookies | ⚠️ Auth + パス | 「JWT cookie auth？」 |
| **error-tracking** | Sentry | ⚠️ パス | 「Sentry使用？」「backend位置？」 |
| **skill-activation-prompt** | ✅ なし | そのままコピー |
| **post-tool-use-tracker** | ✅ なし | そのままコピー |
| **tsc-check** | ⚠️⚠️⚠️ 重い | 「Monorepoまたは単一service？」 |
| **すべてのagents** | ✅ 最小 | パス確認 |
| **すべてのcommands** | ⚠️ パス | 「dev docs位置？」 |

### スキップを推奨する時

| コンポーネント | 以下の場合スキップ... |
|---------------|----------------------|
| **tsc-check hooks** | 単一serviceプロジェクトまたは異なるbuild設定 |
| **route-tester** | JWT cookie認証未使用 |
| **frontend-dev-guidelines** | React + MUI未使用 |
| **auth agents** | JWT cookie auth未使用 |

---

## Claude向け最終Tips

**ユーザーが「すべて追加して」と言う時：**
- 必須要素から開始：skill-activation hooks + 関連skills 1-2個
- すべての5 skills + 10 agentsで圧倒しない
- 実際に必要なものは何か質問

**何かが動作しない時：**
- 検証チェックリストを確認
- パスが構造と一致するか検証
- Hooksを手動でテスト
- JSON構文エラーを確認

**ユーザーが確信がない時：**
- skill-activation hooksだけで開始を推奨
- BackendまたはfrontendのSkillを追加（使用するもの）
- 後で必要に応じて追加

**常に実行内容を説明：**
- 実行中のコマンドを表示
- 質問する理由を説明
- 統合後に明確な次のステップを提供

---

**覚えておいてください：** これはリファレンスライブラリであり、動作するアプリケーションではありません。あなたの役割は、ユーザーが自分の特定のプロジェクト構造に合わせてコンポーネントを選別し調整するのを助けることです。
