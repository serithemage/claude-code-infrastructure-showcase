# Sentry統合とモニタリング

Sentry v8を使用したエラー追跡とパフォーマンスモニタリングの完全なガイドです。

## 目次

- [核心原則](#核心原則)
- [Sentry初期化](#sentry初期化)
- [エラーキャプチャパターン](#エラーキャプチャパターン)
- [パフォーマンスモニタリング](#パフォーマンスモニタリング)
- [Cron Jobモニタリング](#cron-jobモニタリング)
- [エラーコンテキストモベストプラクティス](#エラーコンテキストモベストプラクティス)
- [一般的な間違い](#一般的な間違い)

---

## 核心原則

**必須**: すべてのエラーは必ずSentryにキャプチャされなければなりません。例外なし。

**すべてのエラーはキャプチャされるべき** - すべてのサービスで包括的なエラー追跡とともにSentry v8を使用してください。

---

## Sentry初期化

### instrument.tsパターン

**場所:** `src/instrument.ts`（server.tsとすべてのcron jobsで最初のimportでなければならない）

**マイクロサービス用テンプレート:**

```typescript
import * as Sentry from '@sentry/node';
import * as fs from 'fs';
import * as path from 'path';
import * as ini from 'ini';

const sentryConfigPath = path.join(__dirname, '../sentry.ini');
const sentryConfig = ini.parse(fs.readFileSync(sentryConfigPath, 'utf-8'));

Sentry.init({
    dsn: sentryConfig.sentry?.dsn,
    environment: process.env.NODE_ENV || 'development',
    tracesSampleRate: parseFloat(sentryConfig.sentry?.tracesSampleRate || '0.1'),
    profilesSampleRate: parseFloat(sentryConfig.sentry?.profilesSampleRate || '0.1'),

    integrations: [
        ...Sentry.getDefaultIntegrations({}),
        Sentry.extraErrorDataIntegration({ depth: 5 }),
        Sentry.localVariablesIntegration(),
        Sentry.requestDataIntegration({
            include: {
                cookies: false,
                data: true,
                headers: true,
                ip: true,
                query_string: true,
                url: true,
                user: { id: true, email: true, username: true },
            },
        }),
        Sentry.consoleIntegration(),
        Sentry.contextLinesIntegration(),
        Sentry.prismaIntegration(),
    ],

    beforeSend(event, hint) {
        // ヘルスチェックをフィルタリング
        if (event.request?.url?.includes('/healthcheck')) {
            return null;
        }

        // 機密ヘッダーをスクラブ
        if (event.request?.headers) {
            delete event.request.headers['authorization'];
            delete event.request.headers['cookie'];
        }

        // PIIのためにメールをマスク
        if (event.user?.email) {
            event.user.email = event.user.email.replace(/^(.{2}).*(@.*)$/, '$1***$2');
        }

        return event;
    },

    ignoreErrors: [
        /^Invalid JWT/,
        /^JWT expired/,
        'NetworkError',
    ],
});

// サービスコンテキストを設定
Sentry.setTags({
    service: 'form',
    version: '1.0.1',
});

Sentry.setContext('runtime', {
    node_version: process.version,
    platform: process.platform,
});
```

**重要ポイント:**
- PII保護内蔵（beforeSend）
- 非重要エラーのフィルタリング
- 包括的な統合
- Prisma計装
- サービス別タギング

---

## エラーキャプチャパターン

### 1. BaseControllerパターン

```typescript
// BaseController.handleErrorを使用
protected handleError(error: unknown, res: Response, context: string, statusCode = 500): void {
    Sentry.withScope((scope) => {
        scope.setTag('controller', this.constructor.name);
        scope.setTag('operation', context);
        scope.setUser({ id: res.locals?.claims?.userId });
        Sentry.captureException(error);
    });

    res.status(statusCode).json({
        success: false,
        error: { message: error instanceof Error ? error.message : 'Error occurred' }
    });
}
```

### 2. Workflowエラー処理

```typescript
import { SentryHelper } from '../utils/sentryHelper';

try {
    await businessOperation();
} catch (error) {
    SentryHelper.captureOperationError(error, {
        operationType: 'POST_CREATION',
        entityId: 123,
        userId: 'user-123',
        operation: 'createPost',
    });
    throw error;
}
```

### 3. Serviceレイヤーエラー処理

```typescript
try {
    await someOperation();
} catch (error) {
    Sentry.captureException(error, {
        tags: {
            service: 'form',
            operation: 'someOperation'
        },
        extra: {
            userId: currentUser.id,
            entityId: 123
        }
    });
    throw error;
}
```

---

## パフォーマンスモニタリング

### データベースパフォーマンス追跡

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({ take: 5 });
    }
);
```

### APIエンドポイントSpans

```typescript
router.post('/operation', async (req, res) => {
    return await Sentry.startSpan({
        name: 'operation.execute',
        op: 'http.server',
        attributes: {
            'http.method': 'POST',
            'http.route': '/operation'
        }
    }, async () => {
        const result = await performOperation();
        res.json(result);
    });
});
```

---

## Cron Jobモニタリング

### 必須パターン

```typescript
#!/usr/bin/env node
import '../instrument'; // shebang後の最初の行
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
            // Cron jobロジック
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Cron] Error:', error);
            process.exit(1);
        }
    });
}

main().then(() => {
    console.log('[Cron] Completed successfully');
    process.exit(0);
}).catch((error) => {
    console.error('[Cron] Fatal error:', error);
    process.exit(1);
});
```

---

## エラーコンテキストモベストプラクティス

### リッチコンテキスト例

```typescript
Sentry.withScope((scope) => {
    // ユーザーコンテキスト
    scope.setUser({
        id: user.id,
        email: user.email,
        username: user.username
    });

    // フィルタリングのためのタグ
    scope.setTag('service', 'form');
    scope.setTag('endpoint', req.path);
    scope.setTag('method', req.method);

    // 構造化されたコンテキスト
    scope.setContext('operation', {
        type: 'workflow.complete',
        workflowId: 123,
        stepId: 456
    });

    // タイムラインのためのbreadcrumbs
    scope.addBreadcrumb({
        category: 'workflow',
        message: 'Starting step completion',
        level: 'info',
        data: { stepId: 456 }
    });

    Sentry.captureException(error);
});
```

---

## 一般的な間違い

```typescript
// ❌ エラーを飲み込む
try {
    await riskyOperation();
} catch (error) {
    // サイレントに失敗
}

// ❌ 一般的なエラーメッセージ
throw new Error('Error occurred');

// ❌ 機密データの露出
Sentry.captureException(error, {
    extra: { password: user.password } // 絶対禁止
});

// ❌ asyncエラー処理の欠落
async function bad() {
    fetchData().then(data => processResult(data)); // 処理されない
}

// ✅ 適切なasync処理
async function good() {
    try {
        const data = await fetchData();
        processResult(data);
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [async-and-errors.md](async-and-errors.md)
