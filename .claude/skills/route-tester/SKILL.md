---
name: route-tester
description: cookie ベース認証を使用してプロジェクトで認証された routes をテスト。この skill は API エンドポイントテスト、route 機能検証、認証問題デバッグ時に使用します。test-auth-route.js 使用パターンと mock 認証を含みます。
---

# プロジェクト Route Tester Skill

## 目的
この skill は cookie ベース JWT 認証を使用してプロジェクトで認証された routes をテストするためのパターンを提供します。

## この Skill 使用時点
- 新 API エンドポイントテスト
- 変更後の route 機能検証
- 認証問題デバッグ
- POST/PUT/DELETE 操作テスト
- リクエスト/レスポンスデータ検証

## プロジェクト認証概要

プロジェクトが使用するもの:
- **Keycloak** SSO 用 (realm: yourRealm)
- **Cookie ベース JWT** トークン (Bearer ヘッダーではない)
- **Cookie 名**: `refresh_token`
- **JWT 署名**: `config.ini` の secret 使用

## テスト方法

### 方法 1: test-auth-route.js (推奨)

`test-auth-route.js` スクリプトはすべての認証複雑性を自動的に処理します。

**場所**: `/root/git/your project_pre/scripts/test-auth-route.js`

#### 基本 GET リクエスト

```bash
node scripts/test-auth-route.js http://localhost:3000/blog-api/api/endpoint
```

#### JSON データを含む POST リクエスト

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

#### スクリプトが行うこと

1. Keycloak から refresh token 取得
   - ユーザー名: `testuser`
   - パスワード: `testpassword`
2. `config.ini` の JWT secret でトークン署名
3. cookie ヘッダー生成: `refresh_token=<signed-token>`
4. 認証済みリクエスト実行
5. 手動で再現できる正確な curl コマンド表示

#### スクリプト出力

スクリプトが出力するもの:
- リクエスト詳細
- レスポンスステータスと本文
- 手動再現用 curl コマンド

**参考**: スクリプトが詳細なので出力で実際のレスポンスを探してください。

### 方法 2: トークンを使用した手動 curl

test-auth-route.js 出力の curl コマンド使用:

```bash
# スクリプトが以下を出力します:
# 💡 curl で手動テストするには:
# curl -b "refresh_token=eyJhbGci..." http://localhost:3000/blog-api/api/endpoint

# その curl コマンドをコピーして修正:
curl -X POST http://localhost:3000/blog-api/777/submit \
  -H "Content-Type: application/json" \
  -b "refresh_token=<スクリプト出力からトークンコピー>" \
  -d '{"your": "data"}'
```

### 方法 3: Mock 認証 (開発環境のみ - 最も簡単)

開発環境で mock auth を使用して Keycloak を完全にバイパスします。

#### 設定

```bash
# サービス .env ファイルに追加 (例: blog-api/.env)
MOCK_AUTH=true
MOCK_USER_ID=test-user
MOCK_USER_ROLES=admin,operations
```

#### 使用法

```bash
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-user" \
     -H "X-Mock-Roles: admin,operations" \
     http://localhost:3002/api/protected
```

#### Mock Auth 要件

Mock auth は以下の場合にのみ動作:
- `NODE_ENV` が `development` または `test`
- `mockAuth` middleware が route に追加済み
- プロダクションでは絶対に動作しない (セキュリティ機能)

## 一般的なテストパターン

### Form Submission テスト

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

### Workflow 開始テスト

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/start \
    POST \
    '{"workflowCode":"DHS_CLOSEOUT","entityType":"Submission","entityID":123}'
```

### Workflow Step 完了テスト

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/step/complete \
    POST \
    '{"stepInstanceID":789,"answers":{"decision":"approved","comments":"Looks good"}}'
```

### Query Parameters がある GET テスト

```bash
node scripts/test-auth-route.js \
    "http://localhost:3002/api/workflows?status=active&limit=10"
```

### ファイルアップロードテスト

```bash
# まず test-auth-route.js からトークンを取得後:
curl -X POST http://localhost:5000/upload \
  -H "Content-Type: multipart/form-data" \
  -b "refresh_token=<TOKEN>" \
  -F "file=@/path/to/file.pdf" \
  -F "metadata={\"description\":\"Test file\"}"
```

## ハードコードされたテスト資格情報

`test-auth-route.js` スクリプトが使用する資格情報:

- **ユーザー名**: `testuser`
- **パスワード**: `testpassword`
- **Keycloak URL**: `config.ini` から (通常 `http://localhost:8081`)
- **Realm**: `yourRealm`
- **Client ID**: `config.ini` から

## サービスポート

| サービス | ポート | Base URL |
|---------|------|----------|
| Users   | 3000 | http://localhost:3000 |
| Projects| 3001 | http://localhost:3001 |
| Form    | 3002 | http://localhost:3002 |
| Email   | 3003 | http://localhost:3003 |
| Uploads | 5000 | http://localhost:5000 |

## Route 接頭辞

各サービスの `/src/app.ts` で route 接頭辞を確認:

```typescript
// blog-api/src/app.ts 例
app.use('/blog-api/api', formRoutes);          // 接頭辞: /blog-api/api
app.use('/api/workflow', workflowRoutes);  // 接頭辞: /api/workflow
```

**完全 Route** = Base URL + 接頭辞 + Route パス

例:
- Base: `http://localhost:3002`
- 接頭辞: `/form`
- Route: `/777/submit`
- **完全 URL**: `http://localhost:3000/blog-api/777/submit`

## テストチェックリスト

Route テスト前:

- [ ] サービス識別 (form, email, users など)
- [ ] 正しいポート特定
- [ ] `app.ts` で route 接頭辞確認
- [ ] 完全 URL 構成
- [ ] リクエスト本文準備 (POST/PUT の場合)
- [ ] 認証方法決定
- [ ] テスト実行
- [ ] レスポンスステータスとデータ検証
- [ ] 該当する場合データベース変更確認

## データベース変更検証

データを修正する routes テスト後:

```bash
# MySQL に接続
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev

# 特定テーブル確認
mysql> SELECT * FROM WorkflowInstance WHERE id = 123;
mysql> SELECT * FROM WorkflowStepInstance WHERE instanceId = 123;
mysql> SELECT * FROM WorkflowNotification WHERE recipientUserId = 'user-123';
```

## 失敗したテストのデバッグ

### 401 Unauthorized

**考えられる原因**:
1. トークン期限切れ (test-auth-route.js で再生成)
2. 間違った cookie フォーマット
3. JWT secret 不一致
4. Keycloak が実行中でない

**解決策**:
```bash
# Keycloak が実行中か確認
docker ps | grep keycloak

# トークン再生成
node scripts/test-auth-route.js http://localhost:3002/api/health

# config.ini に正しい jwtSecret があるか確認
```

### 403 Forbidden

**考えられる原因**:
1. ユーザーに必要なロールがない
2. リソース権限が間違っている
3. Route に特定権限が必要

**解決策**:
```bash
# admin ロールで mock auth 使用
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-admin" \
     -H "X-Mock-Roles: admin" \
     http://localhost:3002/api/protected
```

### 404 Not Found

**考えられる原因**:
1. 間違った URL
2. 欠落した route 接頭辞
3. Route が登録されていない

**解決策**:
1. `app.ts` で route 接頭辞確認
2. Route 登録確認
3. サービスが実行中か確認 (`pm2 list`)

### 500 Internal Server Error

**考えられる原因**:
1. データベース接続問題
2. 必須フィールド欠落
3. 検証エラー
4. アプリケーションエラー

**解決策**:
1. サービスログ確認 (`pm2 logs <service>`)
2. Sentry でエラー詳細確認
3. リクエスト本文が予想スキーマと一致するか確認
4. データベース接続確認

## auth-route-tester Agent 使用

変更後の包括的な route テスト用:

1. **影響を受ける routes 識別**
2. **Route 情報収集**:
   - 完全 route パス (接頭辞含む)
   - 予想 POST データ
   - 検証するテーブル
3. **auth-route-tester agent 呼び出し**

Agent が行うこと:
- 適切な認証で route テスト
- データベース変更検証
- レスポンスフォーマット確認
- 問題報告

## 例テストシナリオ

### 新 Route 作成後

```bash
# 1. 有効なデータでテスト
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"value1","field2":"value2"}'

# 2. データベース検証
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev \
    -e "SELECT * FROM MyTable ORDER BY createdAt DESC LIMIT 1;"

# 3. 無効なデータでテスト
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"invalid"}'

# 4. 認証なしでテスト
curl http://localhost:3002/api/my-new-route
# 401 を返すべき
```

### Route 修正後

```bash
# 1. 既存機能がまだ動作するかテスト
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"existing":"data"}'

# 2. 新機能テスト
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"new":"field","existing":"data"}'

# 3. 下位互換性検証
# 以前のリクエスト形式でテスト (該当する場合)
```

## 設定ファイル

### config.ini (各サービス)

```ini
[keycloak]
url = http://localhost:8081
realm = yourRealm
clientId = app-client

[jwt]
jwtSecret = your-jwt-secret-here
```

### .env (各サービス)

```bash
NODE_ENV=development
MOCK_AUTH=true           # 任意: mock auth 有効化
MOCK_USER_ID=test-user   # 任意: デフォルト mock ユーザー
MOCK_USER_ROLES=admin    # 任意: デフォルト mock ロール
```

## 核心ファイル

- `/root/git/your project_pre/scripts/test-auth-route.js` - メインテストスクリプト
- `/blog-api/src/app.ts` - Form service routes
- `/notifications/src/app.ts` - Email service routes
- `/auth/src/app.ts` - Users service routes
- `/config.ini` - サービス設定
- `/.env` - 環境変数

## 関連 Skills

- データベース変更検証に **database-verification** 使用
- キャプチャされたエラー確認に **error-tracking** 使用
- Workflow route テストに **workflow-builder** 使用
- 通知送信確認に **notification-sender** 使用
