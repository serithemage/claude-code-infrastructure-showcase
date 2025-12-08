# Dev Docsパターン

Claude Codeセッションとcontextリセット間でプロジェクトcontextを維持するための方法論です。

---

## 問題点

**Contextリセットはすべてを失います：**
- 実装決定事項
- 重要ファイルとその目的
- 作業進行状況
- 技術的制約
- 特定のアプローチを選んだ理由

**リセット後、Claudeはすべてを再発見しなければなりません。**

---

## 解決策：永続的なDev Docs

作業再開に必要なすべてを含む3ファイル構造：

```
dev/active/[task-name]/
├── [task-name]-plan.md      # 戦略的計画
├── [task-name]-context.md   # 重要決定事項 & ファイル
└── [task-name]-tasks.md     # チェックリスト形式
```

**これらのファイルはcontextリセットでも生き残ります** - Claudeがこれを読んで即座に作業を再開できます。

---

## 3ファイル構造

### 1. [task-name]-plan.md

**目的：** 実装のための戦略的計画

**含まれる内容：**
- 要約
- 現在状態分析
- 提案される将来状態
- 実装ステップ
- 受け入れ基準を含む詳細タスク
- リスク評価
- 成功指標
- スケジュール見積もり

**生成時期：** 複雑なタスク開始時

**更新時期：** スコープが変更されたり新しいステップが発見された時

**例：**
```markdown
# 機能名 - 実装計画

## 要約
何を構築していて、なぜするのか

## 現在状態
現在地

## 実装ステップ

### Phase 1: Infrastructure（2時間）
- Task 1.1: Database schema設定
  - 受け入れ: Schemaコンパイル済み、関係正確
- Task 1.2: Service構造生成
  - 受け入れ: すべてのディレクトリ生成済み

### Phase 2: Core Functionality（3時間）
...
```

---

### 2. [task-name]-context.md

**目的：** 作業再開のための重要情報

**含まれる内容：**
- SESSION PROGRESSセクション（頻繁に更新！）
- 完了したもの vs 進行中のもの
- 重要ファイルとその目的
- 重要な決定事項
- 発見された技術的制約
- 関連ファイルリンク
- 迅速な再開指示

**生成時期：** タスク開始時

**更新時期：** **頻繁に** - 重要な決定、完了、または発見後

**例：**
```markdown
# 機能名 - Context

## SESSION PROGRESS (2025-10-29)

### ✅ 完了
- Database schema生成済み（User、Post、Commentモデル）
- PostControllerをBaseControllerパターンで実装
- Sentry統合動作中

### 🟡 進行中
- ビジネスロジックがあるPostService生成中
- ファイル: src/services/postService.ts

### ⚠️ ブロッカー
- キャッシュ戦略決定必要

## 重要ファイル

**src/controllers/PostController.ts**
- BaseController拡張
- Postsに対するHTTPリクエスト処理
- PostServiceに委任

**src/services/postService.ts**（進行中）
- Post操作のためのビジネスロジック
- 次: キャッシュ追加

## 迅速な再開
続けるには:
1. このファイルを読む
2. PostService.createPost()実装を続ける
3. 残りのタスクはtasksファイル参照
```

**重要：** 重要なタスクが完了するたびにSESSION PROGRESSセクションを更新してください！

---

### 3. [task-name]-tasks.md

**目的：** 進行状況追跡のためのチェックリスト

**含まれる内容：**
- 論理的セクションに分けられたステップ
- チェックボックス形式のタスク
- ステータス表示（✅/🟡/⏳）
- 受け入れ基準
- 迅速な再開セクション

**生成時期：** タスク開始時

**更新時期：** 各タスク完了後または新タスク発見時

**例：**
```markdown
# 機能名 - Taskチェックリスト

## Phase 1: セットアップ ✅ 完了
- [x] Database schema生成
- [x] Controllers設定
- [x] Sentry設定

## Phase 2: 実装 🟡 進行中
- [x] PostController生成
- [ ] PostService生成（進行中）
- [ ] PostRepository生成
- [ ] Zodで検証追加

## Phase 3: テスト ⏳ 未開始
- [ ] Serviceユニットテスト
- [ ] 統合テスト
- [ ] 手動APIテスト
```

---

## Dev Docs使用時期

**使用する場合：**
- ✅ 複数日にわたる複雑なタスク
- ✅ 多くの部分が動く機能
- ✅ 複数セッションにまたがる可能性のある作業
- ✅ 慎重な計画が必要な作業
- ✅ 大規模システムリファクタリング

**スキップする場合：**
- ❌ 簡単なバグ修正
- ❌ 単一ファイル変更
- ❌ 迅速な更新
- ❌ 些細な修正

**経験則：** 2時間以上かかるか複数セッションにまたがるならdev docsを使用。

---

## Dev Docsを使用したワークフロー

### 新タスク開始

1. **/dev-docs slash command使用：**
   ```
   /dev-docs refactor authentication system
   ```

2. **Claudeが3つのファイル生成：**
   - 要件分析
   - コードベース検査
   - 包括的な計画生成
   - Contextおよびtasksファイル生成

3. **レビューと調整：**
   - 計画が合理的か確認
   - 欠落した考慮事項追加
   - スケジュール見積もり調整

### 実装中

1. 全体戦略のために**plan参照**
2. **context.mdを頻繁に更新：**
   - 完了したタスクを表示
   - 決定事項を記録
   - ブロッカーを追加
3. タスクを完了したらtasks.mdで**タスクをチェック**

### Contextリセット後

1. **Claudeが3つのファイルすべて読む**
2. 数秒で**完全な状態理解**
3. **正確に中断したところから再開**

何をしていたか説明する必要なし - すべてドキュメント化されています！

---

## Slash Commandsとの統合

### /dev-docs
**生成：** タスクのための新しいdev docs

**使用法：**
```
/dev-docs implement real-time notifications
```

**生成されるもの：**
- `dev/active/implement-real-time-notifications/`
  - implement-real-time-notifications-plan.md
  - implement-real-time-notifications-context.md
  - implement-real-time-notifications-tasks.md

### /dev-docs-update
**更新：** Contextリセット前の既存dev docs

**使用法：**
```
/dev-docs-update
```

**更新内容：**
- 完了したタスクを表示
- 発見された新タスク追加
- セッション進行状況でcontext更新
- 現在状態をキャプチャ

**使用時期：** Context制限に近づいたときやセッションを終了する時

---

## ファイル構成

```
dev/
├── README.md              # このファイル
├── active/                # 現在のタスク
│   ├── task-1/
│   │   ├── task-1-plan.md
│   │   ├── task-1-context.md
│   │   └── task-1-tasks.md
│   └── task-2/
│       └── ...
└── archive/               # 完了したタスク（オプション）
    └── old-task/
        └── ...
```

**active/**: 進行中のタスク
**archive/**: 完了したタスク（参照用）

---

## 例：実際の使用

このrepositoryの**dev/active/public-infrastructure-repo/**で実際の例を確認：
- **plan.md** - このshowcase作成のための700行以上の戦略的計画
- **context.md** - 完了した内容、決定事項、次にやること追跡
- **tasks.md** - すべてのステップとタスクのチェックリスト

これはこのshowcaseを構築するのに使用された実際のdev docsです！

---

## ベストプラクティス

### Contextを頻繁に更新

**悪い：** セッション終了時のみ更新
**良い：** 各主要マイルストーン後に更新

**SESSION PROGRESSセクションは常に現実を反映すべき：**
```markdown
## SESSION PROGRESS (YYYY-MM-DD)

### ✅ 完了（完了したすべてを列挙）
### 🟡 進行中（今まさに作業中のもの）
### ⚠️ ブロッカー（進行を妨げているもの）
```

### タスクを実行可能にする

**悪い：** 「認証修正」
**良い：** 「AuthMiddleware.tsでJWTトークン検証実装（受け入れ: トークン検証済み、エラーはSentryへ）」

**含めるべき内容：**
- 具体的なファイル名
- 明確な受け入れ基準
- 他のタスクへの依存関係

### 計画を最新状態に維持

スコープが変更されたら：
- 計画を更新
- 新しいステップを追加
- スケジュール見積もりを調整
- スコープが変更された理由を記録

---

## Claude Code向け

**ユーザーがdev docs生成を要求した時：**

1. **可能なら/dev-docs slash commandを使用**
2. **または手動で生成：**
   - タスクスコープについて質問
   - 関連コードベースファイル分析
   - 包括的な計画生成
   - Contextおよびtasks生成

3. **以下を含めて計画を構成：**
   - 明確なステップ
   - 実行可能なタスク
   - 受け入れ基準
   - リスク評価

4. **Contextファイルを再開可能にする：**
   - 一番上にSESSION PROGRESS
   - 迅速な再開指示
   - 説明付きの重要ファイルリスト

**Dev docsから再開する時：**

1. **3つのファイルすべて読む**（plan、context、tasks）
2. **context.mdから開始** - 現在状態がある
3. **tasks.mdを確認** - 完了したものと次にやることを確認
4. **plan.md参照** - 全体戦略を理解

**頻繁に更新：**
- タスク完了したらすぐ表示
- 重要なタスク後SESSION PROGRESSを更新
- 発見された新タスクを追加

---

## Dev Docs手動作成

/dev-docs commandがない場合：

**1. ディレクトリ作成：**
```bash
mkdir -p dev/active/your-task-name
```

**2. plan.md作成：**
- 要約
- 実装ステップ
- 詳細タスク
- スケジュール見積もり

**3. context.md作成：**
- SESSION PROGRESSセクション
- 重要ファイル
- 重要な決定事項
- 迅速な再開指示

**4. tasks.md作成：**
- チェックボックス付きステップ
- [ ] タスク形式
- 受け入れ基準

---

## メリット

**Dev docs以前：**
- Contextリセット = 最初からやり直し
- 決定の理由を忘れる
- 進行状況を失う
- タスク繰り返し

**Dev docs以後：**
- Contextリセット = 3ファイル読んで即再開
- 決定事項ドキュメント化済み
- 進行状況追跡済み
- 繰り返しタスクなし

**節約された時間：** Contextリセットごとに数時間

---

## 次のステップ

1. 次の複雑なタスクで**パターンを試す**
2. **/dev-docs** slash commandを使用（可能な場合）
3. **頻繁に更新** - 特にcontext.md
4. **実際の動作確認** - dev/active/public-infrastructure-repo/を閲覧

**質問がありますか？** [CLAUDE_INTEGRATION_GUIDE.md](../CLAUDE_INTEGRATION_GUIDE.md)を参照してください
