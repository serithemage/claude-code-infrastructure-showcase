# Asyncパターンとエラー処理

async/awaitパターンとカスタムエラー処理の完全なガイドです。

## 目次

- [Async/Awaitベストプラクティス](#asyncawaitベストプラクティス)
- [Promiseエラー処理](#promiseエラー処理)
- [カスタムエラータイプ](#カスタムエラータイプ)
- [asyncErrorWrapperユーティリティ](#asyncerrorwrapperユーティリティ)
- [エラー伝播](#エラー伝播)
- [一般的なAsync落とし穴](#一般的なasync落とし穴)

---

## Async/Awaitベストプラクティス

### 常にTry-Catch使用

```typescript
// ❌ 絶対禁止: 処理されないasyncエラー
async function fetchData() {
    const data = await database.query(); // スローしても処理されない！
    return data;
}

// ✅ 常に: try-catchでラップ
async function fetchData() {
    try {
        const data = await database.query();
        return data;
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

### .then()チェーンを避ける

```typescript
// ❌ 避ける: Promiseチェーン
function processData() {
    return fetchData()
        .then(data => transform(data))
        .then(transformed => save(transformed))
        .catch(error => {
            console.error(error);
        });
}

// ✅ 推奨: Async/await
async function processData() {
    try {
        const data = await fetchData();
        const transformed = await transform(data);
        return await save(transformed);
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

---

## Promiseエラー処理

### 並列操作

```typescript
// ✅ Promise.allでエラー処理
try {
    const [users, profiles, settings] = await Promise.all([
        userService.getAll(),
        profileService.getAll(),
        settingsService.getAll(),
    ]);
} catch (error) {
    // 1つが失敗すると全体が失敗
    Sentry.captureException(error);
    throw error;
}

// ✅ Promise.allSettledで個別エラー処理
const results = await Promise.allSettled([
    userService.getAll(),
    profileService.getAll(),
    settingsService.getAll(),
]);

results.forEach((result, index) => {
    if (result.status === 'rejected') {
        Sentry.captureException(result.reason, {
            tags: { operation: ['users', 'profiles', 'settings'][index] }
        });
    }
});
```

---

## カスタムエラータイプ

### カスタムエラー定義

```typescript
// 基底エラークラス
export class AppError extends Error {
    constructor(
        message: string,
        public code: string,
        public statusCode: number,
        public isOperational: boolean = true
    ) {
        super(message);
        this.name = this.constructor.name;
        Error.captureStackTrace(this, this.constructor);
    }
}

// 特定エラータイプ
export class ValidationError extends AppError {
    constructor(message: string) {
        super(message, 'VALIDATION_ERROR', 400);
    }
}

export class NotFoundError extends AppError {
    constructor(message: string) {
        super(message, 'NOT_FOUND', 404);
    }
}

export class ForbiddenError extends AppError {
    constructor(message: string) {
        super(message, 'FORBIDDEN', 403);
    }
}

export class ConflictError extends AppError {
    constructor(message: string) {
        super(message, 'CONFLICT', 409);
    }
}
```

### 使用法

```typescript
// 特定エラーをスロー
if (!user) {
    throw new NotFoundError('User not found');
}

if (user.age < 18) {
    throw new ValidationError('User must be 18+');
}

// Error boundaryが処理
function errorBoundary(error, req, res, next) {
    if (error instanceof AppError) {
        return res.status(error.statusCode).json({
            error: {
                message: error.message,
                code: error.code
            }
        });
    }

    // 不明なエラー
    Sentry.captureException(error);
    res.status(500).json({ error: { message: 'Internal server error' } });
}
```

---

## asyncErrorWrapperユーティリティ

### パターン

```typescript
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

### 使用法

```typescript
// wrapperなし - エラーが処理されない可能性
router.get('/users', async (req, res) => {
    const users = await userService.getAll(); // スローしても処理されない！
    res.json(users);
});

// wrapper使用 - エラーがキャッチされる
router.get('/users', asyncErrorWrapper(async (req, res) => {
    const users = await userService.getAll();
    res.json(users);
}));
```

---

## エラー伝播

### 適切なエラーチェーン

```typescript
// ✅ スタックを通じてエラーを伝播
async function repositoryMethod() {
    try {
        return await PrismaService.main.user.findMany();
    } catch (error) {
        Sentry.captureException(error, { tags: { layer: 'repository' } });
        throw error; // serviceに伝播
    }
}

async function serviceMethod() {
    try {
        return await repositoryMethod();
    } catch (error) {
        Sentry.captureException(error, { tags: { layer: 'service' } });
        throw error; // controllerに伝播
    }
}

async function controllerMethod(req, res) {
    try {
        const result = await serviceMethod();
        res.json(result);
    } catch (error) {
        this.handleError(error, res, 'controllerMethod'); // 最終ハンドラー
    }
}
```

---

## 一般的なAsync落とし穴

### Fire and Forget（悪いパターン）

```typescript
// ❌ 絶対禁止: Fire and forget
async function processRequest(req, res) {
    sendEmail(user.email); // async実行、エラー処理されない！
    res.json({ success: true });
}

// ✅ 常に: awaitまたは処理
async function processRequest(req, res) {
    try {
        await sendEmail(user.email);
        res.json({ success: true });
    } catch (error) {
        Sentry.captureException(error);
        res.status(500).json({ error: 'Failed to send email' });
    }
}

// ✅ または: 意図的なバックグラウンドタスク
async function processRequest(req, res) {
    sendEmail(user.email).catch(error => {
        Sentry.captureException(error);
    });
    res.json({ success: true });
}
```

### 処理されないRejection

```typescript
// ✅ 処理されないrejectionのためのグローバルハンドラー
process.on('unhandledRejection', (reason, promise) => {
    Sentry.captureException(reason, {
        tags: { type: 'unhandled_rejection' }
    });
    console.error('Unhandled Rejection:', reason);
});

process.on('uncaughtException', (error) => {
    Sentry.captureException(error, {
        tags: { type: 'uncaught_exception' }
    });
    console.error('Uncaught Exception:', error);
    process.exit(1);
});
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md)
- [sentry-and-monitoring.md](sentry-and-monitoring.md)
- [complete-examples.md](complete-examples.md)
