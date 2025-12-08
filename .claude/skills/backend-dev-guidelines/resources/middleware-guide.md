# Middlewareガイド - Express Middlewareパターン

バックエンドマイクロサービスでのmiddleware作成と使用に関する完全なガイドです。

## 目次

- [認証Middleware](#認証middleware)
- [AsyncLocalStorageを使用したAudit Middleware](#asynclocalstorageを使用したaudit-middleware)
- [Error Boundary Middleware](#error-boundary-middleware)
- [Validation Middleware](#validation-middleware)
- [組み合わせ可能なMiddleware](#組み合わせ可能なmiddleware)
- [Middleware順序](#middleware順序)

---

## 認証Middleware

### SSOMiddlewareパターン

**ファイル:** `/form/src/middleware/SSOMiddleware.ts`

```typescript
export class SSOMiddlewareClient {
    static verifyLoginStatus(req: Request, res: Response, next: NextFunction): void {
        const token = req.cookies.refresh_token;

        if (!token) {
            return res.status(401).json({ error: 'Not authenticated' });
        }

        try {
            const decoded = jwt.verify(token, config.tokens.jwt);
            res.locals.claims = decoded;
            res.locals.effectiveUserId = decoded.sub;
            next();
        } catch (error) {
            res.status(401).json({ error: 'Invalid token' });
        }
    }
}
```

---

## AsyncLocalStorageを使用したAudit Middleware

### Blog APIの優れたパターン

**ファイル:** `/form/src/middleware/auditMiddleware.ts`

```typescript
import { AsyncLocalStorage } from 'async_hooks';

export interface AuditContext {
    userId: string;
    userName?: string;
    impersonatedBy?: string;
    sessionId?: string;
    timestamp: Date;
    requestId: string;
}

export const auditContextStorage = new AsyncLocalStorage<AuditContext>();

export function auditMiddleware(req: Request, res: Response, next: NextFunction): void {
    const context: AuditContext = {
        userId: res.locals.effectiveUserId || 'anonymous',
        userName: res.locals.claims?.preferred_username,
        impersonatedBy: res.locals.isImpersonating ? res.locals.originalUserId : undefined,
        timestamp: new Date(),
        requestId: req.id || uuidv4(),
    };

    auditContextStorage.run(context, () => {
        next();
    });
}

// 現在のコンテキストgetter
export function getAuditContext(): AuditContext | null {
    return auditContextStorage.getStore() || null;
}
```

**利点:**
- リクエスト全体にコンテキストを伝播
- すべての関数にコンテキストを渡す必要なし
- services、repositoriesで自動的に利用可能
- タイプセーフなコンテキストアクセス

**Servicesでの使用:**
```typescript
import { getAuditContext } from '../middleware/auditMiddleware';

async function someOperation() {
    const context = getAuditContext();
    console.log('Operation by:', context?.userId);
}
```

---

## Error Boundary Middleware

### 包括的なエラーハンドラー

**ファイル:** `/form/src/middleware/errorBoundary.ts`

```typescript
export function errorBoundary(
    error: Error,
    req: Request,
    res: Response,
    next: NextFunction
): void {
    // ステータスコード決定
    const statusCode = getStatusCodeForError(error);

    // Sentryにキャプチャ
    Sentry.withScope((scope) => {
        scope.setLevel(statusCode >= 500 ? 'error' : 'warning');
        scope.setTag('error_type', error.name);
        scope.setContext('error_details', {
            message: error.message,
            stack: error.stack,
        });
        Sentry.captureException(error);
    });

    // ユーザーフレンドリーなレスポンス
    res.status(statusCode).json({
        success: false,
        error: {
            message: getUserFriendlyMessage(error),
            code: error.name,
        },
        requestId: Sentry.getCurrentScope().getPropagationContext().traceId,
    });
}

// Asyncラッパー
export function asyncErrorWrapper(
    handler: (req: Request, res: Response, next: NextFunction) => Promise<any>
) {
    return async (req: Request, res: Response, next: NextFunction) => {
        try {
            await handler(req, res, next);
        } catch (error) {
            next(error);
        }
    };
}
```

---

## 組み合わせ可能なMiddleware

### withAuthAndAuditパターン

```typescript
export function withAuthAndAudit(...authMiddleware: any[]) {
    return [
        ...authMiddleware,
        auditMiddleware,
    ];
}

// 使用法
router.post('/:formID/submit',
    ...withAuthAndAudit(SSOMiddlewareClient.verifyLoginStatus),
    async (req, res) => controller.submit(req, res)
);
```

---

## Middleware順序

### 重要な順序（必ず従うこと）

```typescript
// 1. Sentry request handler（最初）
app.use(Sentry.Handlers.requestHandler());

// 2. Bodyパース
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 3. Cookieパース
app.use(cookieParser());

// 4. 認証初期化
app.use(SSOMiddleware.initialize());

// 5. ここでRoutesを登録
app.use('/api/users', userRoutes);

// 6. エラーハンドラー（routes以降）
app.use(errorBoundary);

// 7. Sentryエラーハンドラー（最後）
app.use(Sentry.Handlers.errorHandler());
```

**ルール:** エラーハンドラーは必ずすべてのroutes以降に登録すること！

---

**関連ファイル:**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [async-and-errors.md](async-and-errors.md)
