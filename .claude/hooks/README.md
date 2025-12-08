# Hooks

Skills自動アクティベーション、ファイル追跡、検証を可能にするClaude Code hooksです。

---

## Hooksとは？

Hooksは、Claudeワークフローの特定の時点で実行されるスクリプトです：
- **UserPromptSubmit**: ユーザーがプロンプトを送信した時
- **PreToolUse**: ツールが実行される前
- **PostToolUse**: ツールが完了した後
- **Stop**: ユーザーが停止を要求した時

**核心インサイト：** Hooksはプロンプトを修正し、操作をブロックし、状態を追跡できるため、Claude単独ではできない機能を可能にします。

---

## 必須Hooks（ここから始める）

### skill-activation-prompt（UserPromptSubmit）

**目的：** ユーザープロンプトとファイルコンテキストに基づいて関連skillsを自動提案

**動作方式：**
1. `skill-rules.json`を読む
2. ユーザープロンプトをトリガーパターンとマッチング
3. ユーザーが作業中のファイルを確認
4. Claudeのコンテキストにskill提案を注入

**必須の理由：** Skillsを自動アクティベートさせる核心hookです。

**統合方法：**
```bash
# 両ファイルをコピー
cp skill-activation-prompt.sh your-project/.claude/hooks/
cp skill-activation-prompt.ts your-project/.claude/hooks/

# 実行権限付与
chmod +x your-project/.claude/hooks/skill-activation-prompt.sh

# 依存関係インストール
cd your-project/.claude/hooks
npm install
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

**カスタマイズ：** ✅ 不要 - skill-rules.jsonを自動的に読む

---

### post-tool-use-tracker（PostToolUse）

**目的：** セッション間のコンテキストを維持するためにファイル変更を追跡

**動作方式：**
1. Edit/Write/MultiEditツール呼び出しをモニタリング
2. 修正されたファイルを記録
3. コンテキスト管理のためのキャッシュ生成
4. プロジェクト構造自動検出（frontend、backend、packagesなど）

**必須の理由：** Claudeがコードベースのどの部分がアクティブ状態か理解するのを助けます。

**統合方法：**
```bash
# ファイルをコピー
cp post-tool-use-tracker.sh your-project/.claude/hooks/

# 実行権限付与
chmod +x your-project/.claude/hooks/post-tool-use-tracker.sh
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

**カスタマイズ：** ✅ 不要 - 構造を自動検出

---

## オプションHooks（カスタマイズ必要）

### tsc-check（Stop）

**目的：** ユーザーが停止した時TypeScriptコンパイルチェック

**⚠️ 警告：** マルチサービスmonorepo構造用に設定されています

**統合方法：**

**まずこのhookが適切か確認：**
- ✅ 使用適切: マルチサービスTypeScript monorepo
- ❌ スキップ: 単一サービスプロジェクトまたは異なるビルド設定

**使用する場合：**
1. tsc-check.shをコピー
2. **サービス検出部分を修正（約28行目）：**
   ```bash
   # 例のサービスを実際のサービスに置換:
   case "$repo" in
       api|web|auth|payments|...)  # ← 実際のサービス名
   ```
3. settings.jsonに追加する前に手動でテスト

**カスタマイズ：** ⚠️⚠️⚠️ 多く必要

---

### trigger-build-resolver（Stop）

**目的：** コンパイル失敗時にbuild-error-resolver agentを自動実行

**依存関係：** tsc-check hookが正常動作する必要あり

**カスタマイズ：** ✅ 不要（ただしtsc-checkが先に動作する必要あり）

---

## Claude Code向け案内

**ユーザーのためにhooksを設定する時：**

1. **まず[CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を読む**
2. **常に2つの必須hooksから始める**
3. **Stop hooks追加前にまず確認** - 誤設定するとブロックされる可能性
4. **設定後検証：**
   ```bash
   ls -la .claude/hooks/*.sh | grep rwx
   ```

**質問がありますか？** [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)を参照してください
