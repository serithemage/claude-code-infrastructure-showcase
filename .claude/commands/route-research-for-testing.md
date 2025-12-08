---
description: 修正されたルートのマッピングとテスト実行
argument-hint: "[/extra/path …]"
allowed-tools: Bash(cat:*), Bash(awk:*), Bash(grep:*), Bash(sort:*), Bash(xargs:*), Bash(sed:*)
model: sonnet
---

## コンテキスト

このセッションで変更されたルートファイル（自動生成）：

!cat "$CLAUDE_PROJECT_DIR/.claude/tsc-cache"/\*/edited-files.log \
 | awk -F: '{print $2}' \
 | grep '/routes/' \
 | sort -u

ユーザー指定の追加ルート：`$ARGUMENTS`

## 実行する作業

以下のステップを**正確に**従ってください：

1. 自動リストと`$ARGUMENTS`を結合し、重複を削除し、`src/app.ts`に
   定義されたプレフィックスを解決します。
2. 各最終ルートについて、パス、メソッド、予想されるリクエスト/レスポンス形式、
   有効なペイロードと無効なペイロードの例を含むJSONレコードを出力します。
3. **ここで`Task`ツールを呼び出してください**：

```json
{
    "tool": "Task",
    "parameters": {
        "description": "route smoke tests",
        "prompt": "Run the auth-route-tester sub-agent on the JSON above."
    }
}
```
