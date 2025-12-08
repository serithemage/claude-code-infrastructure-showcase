# Hooks 設定ガイド

このガイドでは、プロジェクトでhooksシステムを設定しカスタマイズする方法を説明します。

## クイックスタート設定

### 1. .claude/settings.jsonにHooksを登録

プロジェクトルートに`.claude/settings.json`を作成または更新してください：

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
    ],
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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/error-handling-reminder.sh"
          }
        ]
      }
    ]
  }
}
```

### 2. 依存関係のインストール

```bash
cd .claude/hooks
npm install
```

### 3. 実行権限の設定

```bash
chmod +x .claude/hooks/*.sh
```

## カスタマイズオプション

### プロジェクト構造の検出

デフォルトでhooksは以下のディレクトリパターンを検出します：

**Frontend:** `frontend/`, `client/`, `web/`, `app/`, `ui/`
**Backend:** `backend/`, `server/`, `api/`, `src/`, `services/`
**Database:** `database/`, `prisma/`, `migrations/`
**Monorepo:** `packages/*`, `examples/*`

#### カスタムディレクトリパターンの追加

`.claude/hooks/post-tool-use-tracker.sh`の`detect_repo()`関数を修正してください：

```bash
case "$repo" in
    # ここにカスタムディレクトリを追加
    my-custom-service)
        echo "$repo"
        ;;
    admin-panel)
        echo "$repo"
        ;;
    # ... 既存のパターン
esac
```

### ビルドコマンドの検出

Hooksは以下に基づいてビルドコマンドを自動検出します：
1. "build"スクリプトがある`package.json`の存在
2. パッケージマネージャー (pnpm > npm > yarn)
3. 特殊ケース (Prisma schema)

#### ビルドコマンドのカスタマイズ

`.claude/hooks/post-tool-use-tracker.sh`の`get_build_command()`関数を修正してください：

```bash
# カスタムビルドロジックを追加
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && make build"
    return
fi
```

### TypeScript設定

Hooksは自動的に検出します：
- 標準TypeScriptプロジェクト用の`tsconfig.json`
- Vite/Reactプロジェクト用の`tsconfig.app.json`

#### カスタムTypeScript設定

`.claude/hooks/post-tool-use-tracker.sh`の`get_tsc_command()`関数を修正してください：

```bash
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && npx tsc --project tsconfig.build.json --noEmit"
    return
fi
```

### Prettier設定

Prettier hookは以下の順序で設定を検索します：
1. 現在のファイルディレクトリ（上位へ探索）
2. プロジェクトルート
3. Prettierデフォルトにフォールバック

#### カスタムPrettier設定の検索

`.claude/hooks/stop-prettier-formatter.sh`の`get_prettier_config()`関数を修正してください：

```bash
# カスタム設定場所を追加
if [[ -f "$project_root/config/.prettierrc" ]]; then
    echo "$project_root/config/.prettierrc"
    return
fi
```

### エラー処理リマインダー

`.claude/hooks/error-handling-reminder.ts`でファイルカテゴリ検出を設定してください：

```typescript
function getFileCategory(filePath: string): 'backend' | 'frontend' | 'database' | 'other' {
    // カスタムパターンを追加
    if (filePath.includes('/my-custom-dir/')) return 'backend';
    // ... 既存のパターン
}
```

### エラー閾値の設定

auto-error-resolver agent推奨タイミングを変更します。

`.claude/hooks/stop-build-check-enhanced.sh`を修正してください：

```bash
# デフォルトは5エラー - 希望の値に変更
if [[ $total_errors -ge 10 ]]; then  # 今は10以上のエラーが必要
    # agent推奨
fi
```

## 環境変数

### グローバル環境変数

シェルプロファイル（`.bashrc`、`.zshrc`など）に設定してください：

```bash
# エラー処理リマインダーを無効化
export SKIP_ERROR_REMINDER=1

# カスタムプロジェクトディレクトリ（デフォルトを使用しない場合）
export CLAUDE_PROJECT_DIR=/path/to/your/project
```

### セッション別環境変数

Claude Code起動前に設定してください：

```bash
SKIP_ERROR_REMINDER=1 claude-code
```

## Hook実行順序

Stop hooksは`settings.json`に指定された順序で実行されます：

```json
"Stop": [
  {
    "hooks": [
      { "command": "...formatter.sh" },    // 最初に実行
      { "command": "...build-check.sh" },  // 2番目に実行
      { "command": "...reminder.sh" }      // 3番目に実行
    ]
  }
]
```

**この順序が重要な理由：**
1. まずファイルをフォーマット（きれいなコード）
2. 次にエラーを確認
3. 最後にリマインダーを表示

## 選択的Hook有効化

すべてのhooksが必要なわけではありません。プロジェクトに合ったものを選んでください：

### 最小設定（Skill活性化のみ）

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

### ビルドチェックのみ（フォーマットなし）

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          }
        ]
      }
    ]
  }
}
```

### フォーマットのみ（ビルドチェックなし）

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          }
        ]
      }
    ]
  }
}
```

## キャッシュ管理

### キャッシュの場所

```
$CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session_id]/
```

### 手動キャッシュクリア

```bash
# すべてのキャッシュデータを削除
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/*

# 特定のセッションを削除
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session-id]
```

### 自動クリーンアップ

build-check hookはビルド成功時にセッションキャッシュを自動的にクリーンアップします。

## 設定のトラブルシューティング

### Hookが実行されない

1. **登録を確認：** `.claude/settings.json`にhookがあるか確認
2. **権限を確認：** `chmod +x .claude/hooks/*.sh`を実行
3. **パスを確認：** `$CLAUDE_PROJECT_DIR`が正しく設定されているか確認
4. **TypeScriptを確認：** `cd .claude/hooks && npx tsc`を実行してエラーを確認

### 誤検出（False Positive）

**問題：** Hookが処理すべきでないファイルでトリガーされる

**解決：** 該当hookにスキップ条件を追加：

```bash
# post-tool-use-tracker.sh内で
if [[ "$file_path" =~ /generated/ ]]; then
    exit 0  # 生成されたファイルをスキップ
fi
```

### パフォーマンス問題

**問題：** Hooksが遅い

**解決策：**
1. TypeScriptチェックを変更されたファイルのみに制限
2. より速いパッケージマネージャーを使用 (pnpm > npm)
3. スキップ条件を追加
4. 大きなファイルでPrettierを無効化

```bash
# stop-prettier-formatter.shで大きなファイルをスキップ
file_size=$(wc -c < "$file" 2>/dev/null || echo 0)
if [[ $file_size -gt 100000 ]]; then  # 100KBより大きいファイルをスキップ
    continue
fi
```

### Hooksのデバッグ

任意のhookにデバッグ出力を追加：

```bash
# hookスクリプトの先頭に
set -x  # デバッグモードを有効化

# または特定のデバッグ行を追加
echo "DEBUG: file_path=$file_path" >&2
echo "DEBUG: repo=$repo" >&2
```

Claude Codeログでhook実行を確認してください。

## 高度な設定

### カスタムHookイベントハンドラー

他のイベント用に独自のhooksを作成できます：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/my-custom-bash-guard.sh"
          }
        ]
      }
    ]
  }
}
```

### モノレポ設定

複数のパッケージがあるモノレポの場合：

```bash
# post-tool-use-tracker.shのdetect_repo()内で
case "$repo" in
    packages)
        # パッケージ名を取得
        local package=$(echo "$relative_path" | cut -d'/' -f2)
        if [[ -n "$package" ]]; then
            echo "packages/$package"
        else
            echo "$repo"
        fi
        ;;
esac
```

### Docker/コンテナプロジェクト

ビルドコマンドがコンテナ内で実行される必要がある場合：

```bash
# post-tool-use-tracker.shのget_build_command()内で
if [[ "$repo" == "api" ]]; then
    echo "docker-compose exec api npm run build"
    return
fi
```

## ベストプラクティス

1. **最小限から始める** - hooksを1つずつ有効化
2. **徹底的にテスト** - 変更後にhooksが動作するか確認
3. **カスタマイズを文書化** - カスタムロジックを説明するコメントを追加
4. **バージョン管理** - `.claude/`ディレクトリをgitにコミット
5. **チームの一貫性** - チーム全体で設定を共有

## 参考資料

- [README.md](./README.md) - Hooks概要
- [../../docs/HOOKS_SYSTEM.md](../../docs/HOOKS_SYSTEM.md) - 完全なhooksリファレンス
- [../../docs/SKILLS_SYSTEM.md](../../docs/SKILLS_SYSTEM.md) - Skills統合
