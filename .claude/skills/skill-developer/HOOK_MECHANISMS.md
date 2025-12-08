# Hookメカニズム - 深堀り分析

UserPromptSubmitおよびPreToolUse hooksの動作方式についての技術的な深堀り分析です。

## 目次

- [UserPromptSubmit Hookフロー](#userpromptsubmit-hookフロー)
- [PreToolUse Hookフロー](#pretooluse-hookフロー)
- [終了コード動作（重要）](#終了コード動作重要)
- [セッション状態管理](#セッション状態管理)
- [パフォーマンス考慮事項](#パフォーマンス考慮事項)

---

## UserPromptSubmit Hookフロー

### 実行順序

```
ユーザーがプロンプトを送信
    ↓
.claude/settings.jsonがhookを登録
    ↓
skill-activation-prompt.sh実行
    ↓
npx tsx skill-activation-prompt.ts
    ↓
Hookがstdinを読み取り（プロンプトを含むJSON）
    ↓
skill-rules.jsonをロード
    ↓
キーワード + intentパターンマッチング
    ↓
優先度別にマッチをグループ化（critical → high → medium → low）
    ↓
フォーマットされたメッセージをstdoutに出力
    ↓
stdoutがClaudeのcontextになる（プロンプトの前に注入）
    ↓
Claudeが見るもの: [skill提案] + ユーザープロンプト
```

### 重要なポイント

- **終了コード**: 常に0（許可）
- **stdout**: → Claudeのcontext（システムメッセージとして注入）
- **タイミング**: Claudeがプロンプトを処理する前に実行
- **動作**: ブロックなし、推奨のみ
- **目的**: Claudeが関連skillsを認識するようにする

### 入力形式

```json
{
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.json",
  "cwd": "/root/git/your-project",
  "permission_mode": "normal",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "layoutシステムはどのように動作しますか？"
}
```

### 出力形式（stdoutへ）

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 SKILL ACTIVATION CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 RECOMMENDED SKILLS:
  → project-catalog-developer

ACTION: Use Skill tool BEFORE responding
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Claudeはユーザーのプロンプトを処理する前にこの出力を追加contextとして見ます。

---

## PreToolUse Hookフロー

### 実行順序

```
ClaudeがEdit/Writeツールを呼び出し
    ↓
.claude/settings.jsonがhookを登録（matcher: Edit|Write）
    ↓
skill-verification-guard.sh実行
    ↓
npx tsx skill-verification-guard.ts
    ↓
Hookがstdinを読み取り（tool_name, tool_inputを含むJSON）
    ↓
skill-rules.jsonをロード
    ↓
ファイルパスパターンを確認（globマッチング）
    ↓
コンテンツパターンのためにファイルを読み取り（ファイルが存在する場合）
    ↓
セッション状態を確認（skillが既に使用されたか？）
    ↓
スキップ条件を確認（ファイルマーカー、環境変数）
    ↓
マッチしてスキップされない場合:
  セッション状態を更新（skillが強制済みとしてマーク）
  ブロックメッセージをstderrに出力
  終了コード2で終了（BLOCK）
それ以外:
  終了コード0で終了（ALLOW）
    ↓
ブロックされた場合:
  stderr → Claudeがメッセージを確認
  Edit/Writeツールが実行されない
  Claudeはskillを使用して再試行する必要がある
許可された場合:
  ツールが正常に実行される
```

### 重要なポイント

- **終了コード2**: BLOCK（stderr → Claude）
- **終了コード0**: ALLOW
- **タイミング**: ツール実行前に実行
- **セッショントラッキング**: 同一セッションでの繰り返しブロックを防止
- **Fail open**: エラー時に作業を許可（ワークフロー中断を防止）
- **目的**: コアguardrailsの強制

### 入力形式

```json
{
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.json",
  "cwd": "/root/git/your-project",
  "permission_mode": "normal",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/root/git/your-project/form/src/services/user.ts",
    "old_string": "...",
    "new_string": "..."
  }
}
```

### 出力形式（ブロック時stderrへ）

```
⚠️ BLOCKED - Database Operation Detected

📋 REQUIRED ACTION:
1. Use Skill tool: 'database-verification'
2. Verify ALL table and column names against schema
3. Check database structure with DESCRIBE commands
4. Then retry this edit

Reason: Prevent column name errors in Prisma queries
File: form/src/services/user.ts

💡 TIP: Add '// @skip-validation' comment to skip future checks
```

Claudeがこのメッセージを受け取り、編集を再試行する前にskillを使用する必要があることを理解します。

---

## 終了コード動作（重要）

### 終了コード参照表

| 終了コード | stdout | stderr | ツール実行 | Claudeが見るもの |
|-----------|--------|--------|----------|-----------------|
| 0 (UserPromptSubmit) | → Context | → ユーザーのみ | N/A | stdout内容 |
| 0 (PreToolUse) | → ユーザーのみ | → ユーザーのみ | **進行する** | なし |
| 2 (PreToolUse) | → ユーザーのみ | → **CLAUDE** | **ブロック** | stderr内容 |
| その他 | → ユーザーのみ | → ユーザーのみ | ブロック | なし |

### 終了コード2が重要な理由

これが強制のための核心メカニズムです:

1. PreToolUseでClaudeにメッセージを送る**唯一の方法**
2. stderr内容が「Claudeに自動的にフィードバックされる」
3. Claudeがブロックメッセージを見て何をすべきか理解する
4. ツール実行が防止される
5. Guardrails強制に必須

### 会話フローの例

```
ユーザー: "Prismaで新しいユーザーサービスを追加して"

Claude: "ユーザーサービスを作成します..."
    [form/src/services/user.tsの編集を試行]

PreToolUse Hook: [終了コード2]
    stderr: "⚠️ BLOCKED - Use database-verification"

Claudeがエラーを確認して応答:
    "まずデータベーススキーマを確認する必要があります。"
    [Skillツールを使用: database-verification]
    [カラム名を確認]
    [編集を再試行 - 今回は許可される（セッショントラッキング）]
```

---

## セッション状態管理

### 目的

同一セッションでの繰り返し通知を防止 - Claudeがskillを使用すれば再度ブロックしない。

### 状態ファイルの場所

`.claude/hooks/state/skills-used-{session_id}.json`

### 状態ファイルの構造

```json
{
  "skills_used": [
    "database-verification",
    "error-tracking"
  ],
  "files_verified": []
}
```

### 動作方式

1. **最初の編集**（Prismaを含むファイル）:
   - Hookが終了コード2でブロック
   - セッション状態を更新: skills_usedに"database-verification"を追加
   - Claudeがメッセージを見てskillを使用

2. **2回目の編集**（同一セッション）:
   - Hookがセッション状態を確認
   - skills_usedで"database-verification"を発見
   - 終了コード0で終了（許可）
   - Claudeにメッセージなし

3. **別のセッション**:
   - 新しいセッションID = 新しい状態ファイル
   - Hookが再度ブロック

### 制限事項

Hookはskillが*実際に*呼び出されたかどうか検出できません - セッションごと、skillごとに1回だけブロックします。これは以下を意味します:

- Claudeがskillを使用せずに別の編集をしても再度ブロックしない
- Claudeが指示に従うと信頼
- 将来の改善: 実際のSkillツール使用の検出

---

## パフォーマンス考慮事項

### 目標指標

- **UserPromptSubmit**: < 100ms
- **PreToolUse**: < 200ms

### パフォーマンスボトルネック

1. **skill-rules.jsonのロード**（毎回実行時）
   - 将来: メモリにキャッシュ
   - 将来: 変更を監視、必要時のみ再ロード

2. **ファイル内容の読み取り**（PreToolUse）
   - contentPatternsが設定されている場合のみ
   - ファイルが存在する場合のみ
   - 大きなファイルの場合遅くなる可能性

3. **Globマッチング**（PreToolUse）
   - 各パターンに対するRegexコンパイル
   - 将来: 一度コンパイル、キャッシュ

4. **Regexマッチング**（両方のhooks）
   - Intentパターン（UserPromptSubmit）
   - コンテンツパターン（PreToolUse）
   - 将来: 遅延コンパイル、コンパイル済みregexをキャッシュ

### 最適化戦略

**パターンを減らす:**
- より具体的なパターンを使用（確認項目の削減）
- 可能な場合は類似パターンを結合

**ファイルパスパターン:**
- より具体的 = 確認するファイルが減少
- 例: `form/**`より`form/src/services/**`の方が良い

**コンテンツパターン:**
- 本当に必要な場合のみ追加
- よりシンプルなregex = より高速なマッチング

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインskillガイド
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Hook問題のデバッグ
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 設定リファレンス
