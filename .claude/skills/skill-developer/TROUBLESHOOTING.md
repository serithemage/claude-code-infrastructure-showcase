# トラブルシューティング - Skill活性化問題

Skill活性化問題のための包括的なデバッグガイドです。

## 目次

- [Skillがトリガーされない](#skillがトリガーされない)
  - [UserPromptSubmitが提案しない](#userpromptsubmitが提案しない)
  - [PreToolUseがブロックしない](#pretooluse がブロックしない)
- [誤検知](#誤検知)
- [Hookが実行されない](#hookが実行されない)
- [パフォーマンス問題](#パフォーマンス問題)

---

## Skillがトリガーされない

### UserPromptSubmitが提案しない

**症状:** 質問をしたが出力にskill提案が表示されない。

**一般的な原因:**

####  1. キーワードがマッチしない

**確認:**
- skill-rules.jsonの`promptTriggers.keywords`を確認
- プロンプトに実際にキーワードがあるか？
- 注意: 大文字小文字を区別しない部分文字列マッチング

**例:**
```json
"keywords": ["layout", "grid"]
```
- "layoutはどのように動作しますか？" → ✅ "layout"がマッチ
- "gridシステムはどのように動作しますか？" → ✅ "grid"がマッチ
- "layoutsはどのように動作しますか？" → ✅ "layout"がマッチ
- "どのように動作しますか？" → ❌ マッチなし

**解決:** skill-rules.jsonにより多くのキーワードバリエーションを追加

#### 2. Intentパターンが具体的すぎる

**確認:**
- `promptTriggers.intentPatterns`を確認
- https://regex101.com/でregexをテスト
- より広いパターンが必要かもしれない

**例:**
```json
"intentPatterns": [
  "(create|add).*?(database.*?table)"  // 具体的すぎる
]
```
- "database tableを作成して" → ✅ マッチ
- "新しいtableを追加して" → ❌ マッチしない（"database"がない）

**解決:** パターンを広げる:
```json
"intentPatterns": [
  "(create|add).*?(table|database)"  // 改善
]
```

#### 3. Skill名のタイポ

**確認:**
- SKILL.md frontmatterのskill名
- skill-rules.jsonのskill名
- 正確に一致している必要がある

**例:**
```yaml
# SKILL.md
name: project-catalog-developer
```
```json
// skill-rules.json
"project-catalogue-developer": {  // ❌ タイポ: catalogue vs catalog
  ...
}
```

**解決:** 名前が正確に一致するように修正

#### 4. JSON構文エラー

**確認:**
```bash
cat .claude/skills/skill-rules.json | jq .
```

無効なJSONの場合、jqがエラーを表示します。

**一般的なエラー:**
- 末尾のカンマ
- 引用符の欠落
- ダブルクォートの代わりにシングルクォート
- 文字列内のエスケープされていない文字

**解決:** JSON構文を修正、jqで検証

#### デバッグコマンド

手動でhookをテスト:

```bash
echo '{"session_id":"debug","prompt":"ここにテストプロンプト"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

期待: 出力にskillが表示される。

---

### PreToolUseがブロックしない

**症状:** guardrailをトリガーすべきファイルを編集したがブロックが発生しない。

**一般的な原因:**

#### 1. ファイルパスがパターンとマッチしない

**確認:**
- 編集中のファイルパス
- skill-rules.jsonの`fileTriggers.pathPatterns`
- Globパターン構文

**例:**
```json
"pathPatterns": [
  "frontend/src/**/*.tsx"
]
```
- 編集中: `frontend/src/components/Dashboard.tsx` → ✅ マッチ
- 編集中: `frontend/tests/Dashboard.test.tsx` → ✅ マッチ（除外を追加すべき！）
- 編集中: `backend/src/app.ts` → ❌ マッチなし

**解決:** globパターンを調整または欠落パスを追加

#### 2. pathExclusionsで除外されている

**確認:**
- テストファイルを編集していないか？
- `fileTriggers.pathExclusions`を確認

**例:**
```json
"pathExclusions": [
  "**/*.test.ts",
  "**/*.spec.ts"
]
```
- 編集中: `services/user.test.ts` → ❌ 除外される
- 編集中: `services/user.ts` → ✅ 除外されない

**解決:** テスト除外が広すぎる場合は狭めるか削除

#### 3. コンテンツパターンが見つからない

**確認:**
- ファイルに実際にパターンが含まれているか？
- `fileTriggers.contentPatterns`を確認
- Regexが正しいか？

**例:**
```json
"contentPatterns": [
  "import.*[Pp]risma"
]
```
- ファイル内容: `import { PrismaService } from './prisma'` → ✅ マッチ
- ファイル内容: `import { Database } from './db'` → ❌ マッチなし

**デバッグ:**
```bash
# ファイルにパターンがあるか確認
grep -i "prisma" path/to/file.ts
```

**解決:** コンテンツパターンを調整または欠落importを追加

#### 4. セッションで既にSkillが使用されている

**セッション状態を確認:**
```bash
ls .claude/hooks/state/
cat .claude/hooks/state/skills-used-{session-id}.json
```

**例:**
```json
{
  "skills_used": ["database-verification"],
  "files_verified": []
}
```

skillが`skills_used`にある場合、このセッションでは再度ブロックしない。

**解決:** リセットするには状態ファイルを削除:
```bash
rm .claude/hooks/state/skills-used-{session-id}.json
```

#### 5. ファイルマーカーが存在する

**ファイルでスキップマーカーを確認:**
```bash
grep "@skip-validation" path/to/file.ts
```

見つかった場合、ファイルは永久にスキップされる。

**解決:** 検証が再度必要な場合はマーカーを削除

#### 6. 環境変数のオーバーライド

**確認:**
```bash
echo $SKIP_DB_VERIFICATION
echo $SKIP_SKILL_GUARDRAILS
```

設定されている場合、skillは無効化される。

**解決:** 環境変数の設定を解除:
```bash
unset SKIP_DB_VERIFICATION
```

#### デバッグコマンド

手動でhookをテスト:

```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts 2>&1
{
  "session_id": "debug",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/root/git/your-project/form/src/services/user.ts"}
}
EOF
echo "Exit code: $?"
```

期待:
- ブロックすべき場合は終了コード2 + stderrメッセージ
- 許可すべき場合は終了コード0 + 出力なし

---

## 誤検知

**症状:** Skillがトリガーされるべきでないときにトリガーされる。

**一般的な原因と解決策:**

### 1. キーワードが一般的すぎる

**問題:**
```json
"keywords": ["user", "system", "create"]  // 広すぎる
```
- トリガー: "user manual", "file system", "create directory"

**解決:** キーワードをより具体的に
```json
"keywords": [
  "user authentication",
  "user tracking",
  "create feature"
]
```

### 2. Intentパターンが広すぎる

**問題:**
```json
"intentPatterns": [
  "(create)"  // "create"を含むすべてにマッチ
]
```
- トリガー: "create file", "create folder", "create account"

**解決:** パターンにcontextを追加
```json
"intentPatterns": [
  "(create|add).*?(database|table|feature)"  // より具体的
]
```

**高度:** 否定先読みで除外
```regex
(create)(?!.*test).*?(feature)  // "test"が出現したらマッチしない
```

### 3. ファイルパスが一般的すぎる

**問題:**
```json
"pathPatterns": [
  "form/**"  // form/内のすべてにマッチ
]
```
- トリガー: テストファイル、設定ファイル、すべて

**解決:** より狭いパターンを使用
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // サービスファイルのみ
  "form/src/controllers/**/*.ts"
]
```

### 4. コンテンツパターンが関連のないコードを検出

**問題:**
```json
"contentPatterns": [
  "Prisma"  // コメント、文字列などでマッチ
]
```
- トリガー: `// ここでPrismaを使用しないでください`
- トリガー: `const note = "Prisma is cool"`

**解決:** パターンをより具体的に
```json
"contentPatterns": [
  "import.*[Pp]risma",        // importのみ
  "PrismaService\\.",         // 実際の使用のみ
  "prisma\\.(findMany|create)" // 特定のメソッド
]
```

### 5. 適用レベルの調整

**最後の手段:** 誤検知が頻繁な場合:

```json
{
  "enforcement": "block"  // "suggest"に変更
}
```

ブロックの代わりに推奨に変更される。

---

## Hookが実行されない

**症状:** Hookが全く実行されない - 提案もブロックもない。

**一般的な原因:**

### 1. Hookが登録されていない

**`.claude/settings.json`を確認:**
```bash
cat .claude/settings.json | jq '.hooks.UserPromptSubmit'
cat .claude/settings.json | jq '.hooks.PreToolUse'
```

期待: Hookエントリがあるはず

**解決:** 欠落したhook登録を追加:
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

### 2. Bashラッパーが実行可能でない

**確認:**
```bash
ls -l .claude/hooks/*.sh
```

期待: `-rwxr-xr-x`（実行可能）

**解決:**
```bash
chmod +x .claude/hooks/*.sh
```

### 3. 不正なShebang

**確認:**
```bash
head -1 .claude/hooks/skill-activation-prompt.sh
```

期待: `#!/bin/bash`

**解決:** 最初の行に正しいshebangを追加

### 4. npx/tsxが使用できない

**確認:**
```bash
npx tsx --version
```

期待: バージョン番号

**解決:** 依存関係をインストール:
```bash
cd .claude/hooks
npm install
```

### 5. TypeScriptコンパイルエラー

**確認:**
```bash
cd .claude/hooks
npx tsc --noEmit skill-activation-prompt.ts
```

期待: 出力なし（エラーなし）

**解決:** TypeScript構文エラーを修正

---

## パフォーマンス問題

**症状:** Hooksが遅い、プロンプト/編集前に目に見える遅延。

**一般的な原因:**

### 1. パターンが多すぎる

**確認:**
- skill-rules.jsonのパターン数
- 各パターン = regexコンパイル + マッチング

**解決:** パターンを減らす
- 類似パターンを結合
- 重複パターンを削除
- より具体的なパターンを使用（より高速なマッチング）

### 2. 複雑なRegex

**問題:**
```regex
(create|add|modify|update|implement|build).*?(feature|endpoint|route|service|controller|component|UI|page)
```
- 長い代替 = 遅い

**解決:** 簡略化
```regex
(create|add).*?(feature|endpoint)  // より少ない代替
```

### 3. 確認するファイルが多すぎる

**問題:**
```json
"pathPatterns": [
  "**/*.ts"  // すべてのTypeScriptファイルを確認
]
```

**解決:** より具体的に
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 特定のディレクトリのみ
  "form/src/controllers/**/*.ts"
]
```

### 4. 大きなファイル

コンテンツパターンマッチングはファイル全体を読み取る - 大きなファイルの場合遅い。

**解決:**
- 必要な場合のみコンテンツパターンを使用
- ファイルサイズ制限を検討（今後の改善）

### パフォーマンス測定

```bash
# UserPromptSubmit
time echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
time cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

**目標指標:**
- UserPromptSubmit: < 100ms
- PreToolUse: < 200ms

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインskillガイド
- [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md) - Hook動作方式
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 設定リファレンス
