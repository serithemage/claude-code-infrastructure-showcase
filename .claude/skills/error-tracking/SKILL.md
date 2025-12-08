---
name: error-tracking
description: プロジェクトサービスに Sentry v8 error tracking および performance monitoring 追加。この skill は error handling 追加、新 controllers 生成、cron jobs 計測、データベースパフォーマンス追跡時に使用します。すべてのエラーは必ず Sentry にキャプチャされる必要があります - 例外なし。
---

# プロジェクト Sentry 統合 Skill

## 目的
この skill は Sentry v8 パターンに従ってすべてのプロジェクトサービスで包括的な Sentry error tracking および performance monitoring を強制します。

## この Skill 使用時点
- どのコードにでも error handling 追加
- 新 controllers または routes 生成
- Cron jobs 計測
- データベースパフォーマンス追跡
- Performance spans 追加
- Workflow errors 処理

## 🚨 核心ルール

**すべてのエラーは必ず Sentry にキャプチャされる必要があります** - 例外なし。console.error のみを単独で使用しないでください。

## 現在の状態

### Form Service ✅ 完了
- Sentry v8 完全統合
- すべての workflow errors 追跡済み
- SystemActionQueueProcessor 計測済み
- テストエンドポイント利用可能

### Email Service 🟡 進行中
- Phase 1-2 完了 (6/22 作業)
- 189個の ErrorLogger.log() 呼び出し残り

## Sentry 統合パターン

### 1. Controller Error Handling

```typescript
// ✅ 正しい - BaseController 使用
import { BaseController } from '../controllers/BaseController';

export class MyController extends BaseController {
    async myMethod() {
        try {
            // ... コード
        } catch (error) {
            this.handleError(error, 'myMethod'); // 自動的に Sentry に送信
        }
    }
}
```

### 2. Route Error Handling (BaseController なし)

```typescript
import * as Sentry from '@sentry/node';

router.get('/route', async (req, res) => {
    try {
        // ... コード
    } catch (error) {
        Sentry.captureException(error, {
            tags: { route: '/route', method: 'GET' },
            extra: { userId: req.user?.id }
        });
        res.status(500).json({ error: 'Internal server error' });
    }
});
```

### 3. Workflow Error Handling

```typescript
import { WorkflowSentryHelper } from '../workflow/utils/sentryHelper';

// ✅ 正しい - WorkflowSentryHelper 使用
WorkflowSentryHelper.captureWorkflowError(error, {
    workflowCode: 'DHS_CLOSEOUT',
    instanceId: 123,
    stepId: 456,
    userId: 'user-123',
    operation: 'stepCompletion',
    metadata: { additionalInfo: 'value' }
});
```

### 4. Cron Jobs (必須パターン)

```typescript
#!/usr/bin/env node
// shebang 次の最初の行 - 重要！
import '../instrument';
import * as Sentry from '@sentry/node';

async function main() {
    return await Sentry.startSpan({
        name: 'cron.job-name',
        op: 'cron',
        attributes: {
            'cron.job': 'job-name',
            'cron.startTime': new Date().toISOString(),
        }
    }, async () => {
        try {
            // cron job ロジック
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Job] Error:', error);
            process.exit(1);
        }
    });
}

main()
    .then(() => {
        console.log('[Job] 正常に完了');
        process.exit(0);
    })
    .catch((error) => {
        console.error('[Job] Fatal error:', error);
        process.exit(1);
    });
```

### 5. Database Performance Monitoring

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

// ✅ 正しい - データベース操作をラップ
const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({
            take: 5,
        });
    }
);
```

### 6. Spans を使用した Async 作業

```typescript
import * as Sentry from '@sentry/node';

const result = await Sentry.startSpan({
    name: 'operation.name',
    op: 'operation.type',
    attributes: {
        'custom.attribute': 'value'
    }
}, async () => {
    // async 作業
    return await someAsyncOperation();
});
```

## エラーレベル

適切な重大度レベルを使用:

- **fatal**: システム使用不可 (データベースダウン、コアサービス障害)
- **error**: 操作失敗、即座の注意が必要
- **warning**: 回復可能な問題、パフォーマンス低下
- **info**: 情報メッセージ、正常な操作
- **debug**: 詳細デバッグ情報 (開発環境のみ)

## 必須 Context

```typescript
import * as Sentry from '@sentry/node';

Sentry.withScope((scope) => {
    // 使用可能な場合は常に含める
    scope.setUser({ id: userId });
    scope.setTag('service', 'form'); // または 'email', 'users' など
    scope.setTag('environment', process.env.NODE_ENV);

    // 操作別 context 追加
    scope.setContext('operation', {
        type: 'workflow.start',
        workflowCode: 'DHS_CLOSEOUT',
        entityId: 123
    });

    Sentry.captureException(error);
});
```

## サービス別統合

### Form Service

**場所**: `./blog-api/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**主要 Helper**:
- `WorkflowSentryHelper` - Workflow 専用エラー
- `DatabasePerformanceMonitor` - DB クエリ追跡
- `BaseController` - Controller error handling

### Email Service

**場所**: `./notifications/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**主要 Helper**:
- `EmailSentryHelper` - Email 専用エラー
- `BaseController` - Controller error handling

## 設定 (config.ini)

```ini
[sentry]
dsn = your-sentry-dsn
environment = development
tracesSampleRate = 0.1
profilesSampleRate = 0.1

[databaseMonitoring]
enableDbTracing = true
slowQueryThreshold = 100
logDbQueries = false
dbErrorCapture = true
enableN1Detection = true
```

## Sentry 統合テスト

### Form Service テストエンドポイント

```bash
# 基本 error capture テスト
curl http://localhost:3002/blog-api/api/sentry/test-error

# Workflow error テスト
curl http://localhost:3002/blog-api/api/sentry/test-workflow-error

# Database performance テスト
curl http://localhost:3002/blog-api/api/sentry/test-database-performance

# Error boundary テスト
curl http://localhost:3002/blog-api/api/sentry/test-error-boundary
```

### Email Service テストエンドポイント

```bash
# 基本 error capture テスト
curl http://localhost:3003/notifications/api/sentry/test-error

# Email 専用 error テスト
curl http://localhost:3003/notifications/api/sentry/test-email-error

# Performance tracking テスト
curl http://localhost:3003/notifications/api/sentry/test-performance
```

## Performance Monitoring

### 要件

1. **すべての API エンドポイント**はトランザクション追跡必須
2. **100ms 超過データベースクエリ**は自動的にフラグ付け
3. **N+1 クエリ**は検知および報告
4. **Cron jobs**は実行時間追跡必須

### Transaction Tracking

```typescript
import * as Sentry from '@sentry/node';

// Express routes 用自動トランザクション追跡
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// カスタム操作用手動トランザクション
const transaction = Sentry.startTransaction({
    op: 'operation.type',
    name: 'Operation Name',
});

try {
    // 操作実行
} finally {
    transaction.finish();
}
```

## 避けるべき一般的な間違い

❌ **絶対** Sentry なしで console.error だけを使用しないでください
❌ **絶対** エラーを静かに飲み込まないでください
❌ **絶対** エラー context に機密データを露出しないでください
❌ **絶対** context なしで一般的なエラーメッセージを使用しないでください
❌ **絶対** async 操作で error handling をスキップしないでください
❌ **絶対** cron jobs で instrument.ts を最初の行に import することを忘れないでください

## 実装チェックリスト

新コードに Sentry 追加時:

- [ ] Sentry または適切な helper がインポート済み
- [ ] すべての try/catch ブロックが Sentry にキャプチャ
- [ ] エラーに意味のある context 追加済み
- [ ] 適切なエラーレベル使用済み
- [ ] エラーメッセージに機密データなし
- [ ] 遅い操作に performance tracking 追加済み
- [ ] Error handling パステスト済み
- [ ] Cron jobs の場合: instrument.ts が最初にインポート済み

## 核心ファイル

### Form Service
- `/blog-api/src/instrument.ts` - Sentry 初期化
- `/blog-api/src/workflow/utils/sentryHelper.ts` - Workflow エラー
- `/blog-api/src/utils/databasePerformance.ts` - DB モニタリング
- `/blog-api/src/controllers/BaseController.ts` - Controller ベース

### Email Service
- `/notifications/src/instrument.ts` - Sentry 初期化
- `/notifications/src/utils/EmailSentryHelper.ts` - Email エラー
- `/notifications/src/controllers/BaseController.ts` - Controller ベース

### 設定
- `/blog-api/config.ini` - Form service 設定
- `/notifications/config.ini` - Email service 設定
- `/sentry.ini` - 共有 Sentry 設定

## ドキュメント

- 完全実装: `/dev/active/email-sentry-integration/`
- Form service ドキュメント: `/blog-api/docs/sentry-integration.md`
- Email service ドキュメント: `/notifications/docs/sentry-integration.md`

## 関連 Skills

- データベース操作前に **database-verification** 使用
- Workflow error context 用 **workflow-builder** 使用
- データベース error handling 用 **database-scripts** 使用
