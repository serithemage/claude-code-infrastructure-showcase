# RoutingとControllers - モベストプラクティス

クリーンなルート定義とcontrollerパターンの完全なガイドです。

## 目次

- [Routes: ルーティングのみ](#routes-ルーティングのみ)
- [BaseControllerパターン](#basecontrollerパターン)
- [良い例](#良い例)
- [アンチパターン](#アンチパターン)
- [リファクタリングガイド](#リファクタリングガイド)
- [エラー処理](#エラー処理)
- [HTTPステータスコード](#httpステータスコード)

---

## Routes: ルーティングのみ

### 黄金ルール

**Routesは次のことのみを行うべき:**
- ✅ ルートパス定義
- ✅ Middleware登録
- ✅ Controllersへの委譲

**Routesは絶対に行うべきでない:**
- ❌ ビジネスロジックの含有
- ❌ データベースへの直接アクセス
- ❌ バリデーションロジックの実装（Zod + controller使用）
- ❌ 複雑なレスポンスフォーマット
- ❌ 複雑なエラーシナリオの処理

### クリーンなRouteパターン

```typescript
// routes/userRoutes.ts
import { Router } from 'express';
import { UserController } from '../controllers/UserController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';

const router = Router();
const controller = new UserController();

// ✅ クリーン: ルート定義のみ
router.get('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.getUser(req, res)
);

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createUser(req, res)
);

router.put('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.updateUser(req, res)
);

export default router;
```

**キーポイント:**
- 各route: method、path、middlewareチェーン、controller委譲
- try-catch不要（controllerがエラー処理）
- クリーンで、読みやすく、保守しやすい
- 一目ですべてのエンドポイントを把握可能

---

## BaseControllerパターン

### BaseControllerを使用する理由

**利点:**
- すべてのcontrollersで一貫したエラー処理
- 自動Sentry統合
- 標準化されたレスポンスフォーマット
- 再利用可能なヘルパーメソッド
- パフォーマンス追跡ユーティリティ
- ロギングとbreadcrumbヘルパー

### BaseControllerパターン（テンプレート）

**ファイル:** `/email/src/controllers/BaseController.ts`

```typescript
import * as Sentry from '@sentry/node';
import { Response } from 'express';

export abstract class BaseController {
    /**
     * Sentry統合でエラー処理
     */
    protected handleError(
        error: unknown,
        res: Response,
        context: string,
        statusCode = 500
    ): void {
        Sentry.withScope((scope) => {
            scope.setTag('controller', this.constructor.name);
            scope.setTag('operation', context);
            scope.setUser({ id: res.locals?.claims?.userId });

            if (error instanceof Error) {
                scope.setContext('error_details', {
                    message: error.message,
                    stack: error.stack,
                });
            }

            Sentry.captureException(error);
        });

        res.status(statusCode).json({
            success: false,
            error: {
                message: error instanceof Error ? error.message : 'An error occurred',
                code: statusCode,
            },
        });
    }

    /**
     * 成功レスポンス処理
     */
    protected handleSuccess<T>(
        res: Response,
        data: T,
        message?: string,
        statusCode = 200
    ): void {
        res.status(statusCode).json({
            success: true,
            message,
            data,
        });
    }

    /**
     * パフォーマンス追跡ラッパー
     */
    protected async withTransaction<T>(
        name: string,
        operation: string,
        callback: () => Promise<T>
    ): Promise<T> {
        return await Sentry.startSpan(
            { name, op: operation },
            callback
        );
    }

    /**
     * 必須フィールドバリデーション
     */
    protected validateRequest(
        required: string[],
        actual: Record<string, any>,
        res: Response
    ): boolean {
        const missing = required.filter((field) => !actual[field]);

        if (missing.length > 0) {
            Sentry.captureMessage(
                `Missing required fields: ${missing.join(', ')}`,
                'warning'
            );

            res.status(400).json({
                success: false,
                error: {
                    message: 'Missing required fields',
                    code: 'VALIDATION_ERROR',
                    details: { missing },
                },
            });
            return false;
        }
        return true;
    }

    /**
     * ロギングヘルパー
     */
    protected logInfo(message: string, context?: Record<string, any>): void {
        Sentry.addBreadcrumb({
            category: this.constructor.name,
            message,
            level: 'info',
            data: context,
        });
    }

    protected logWarning(message: string, context?: Record<string, any>): void {
        Sentry.captureMessage(message, {
            level: 'warning',
            tags: { controller: this.constructor.name },
            extra: context,
        });
    }

    /**
     * Sentry breadcrumb追加
     */
    protected addBreadcrumb(
        message: string,
        category: string,
        data?: Record<string, any>
    ): void {
        Sentry.addBreadcrumb({ message, category, level: 'info', data });
    }

    /**
     * カスタムメトリクスキャプチャ
     */
    protected captureMetric(name: string, value: number, unit: string): void {
        Sentry.metrics.gauge(name, value, { unit });
    }
}
```

### BaseControllerの使用

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema } from '../validators/userSchemas';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async getUser(req: Request, res: Response): Promise<void> {
        try {
            this.addBreadcrumb('Fetching user', 'user_controller', { userId: req.params.id });

            const user = await this.userService.findById(req.params.id);

            if (!user) {
                return this.handleError(
                    new Error('User not found'),
                    res,
                    'getUser',
                    404
                );
            }

            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // 入力バリデーション
            const validated = createUserSchema.parse(req.body);

            // パフォーマンス追跡
            const user = await this.withTransaction(
                'user.create',
                'db.query',
                () => this.userService.create(validated)
            );

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            const validated = updateUserSchema.parse(req.body);
            const user = await this.userService.update(req.params.id, validated);
            this.handleSuccess(res, user, 'User updated');
        } catch (error) {
            this.handleError(error, res, 'updateUser');
        }
    }
}
```

**利点:**
- 一貫したエラー処理
- 自動Sentry統合
- パフォーマンス追跡
- クリーンで読みやすいコード
- テストしやすい

---

## 良い例

### 例1: Email Notification Routes（優れている ✅）

**ファイル:** `/email/src/routes/notificationRoutes.ts`

```typescript
import { Router } from 'express';
import { NotificationController } from '../controllers/NotificationController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';

const router = Router();
const controller = new NotificationController();

// ✅ 優れている: クリーンな委譲
router.get('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.getNotifications(req, res)
);

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.createNotification(req, res)
);

router.put('/:id/read',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.markAsRead(req, res)
);

export default router;
```

**優れている理由:**
- Routesにビジネスロジックなし
- 明確なmiddlewareチェーン
- 一貫したパターン
- 理解しやすい

### 例2: バリデーション付きProxy Routes（良い ✅）

**ファイル:** `/form/src/routes/proxyRoutes.ts`

```typescript
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => {
        try {
            const validated = createProxySchema.parse(req.body);
            const proxy = await proxyService.createProxyRelationship(validated);
            res.status(201).json({ success: true, data: proxy });
        } catch (error) {
            handler.handleException(res, error);
        }
    }
);
```

**良い理由:**
- Zodバリデーション
- Serviceへの委譲
- 適切なHTTPステータスコード
- エラー処理

**改善できる点:**
- バリデーションをcontrollerに移動
- BaseController使用

---

## アンチパターン

### アンチパターン1: Routesにビジネスロジック（悪い ❌）

**ファイル:** `/form/src/routes/responseRoutes.ts`（実際のプロダクションコード）

```typescript
// ❌ アンチパターン: routeに200行以上のビジネスロジック
router.post('/:formID/submit', async (req: Request, res: Response) => {
    try {
        const username = res.locals.claims.preferred_username;
        const responses = req.body.responses;
        const stepInstanceId = req.body.stepInstanceId;

        // ❌ Routeで権限確認
        const userId = await userProfileService.getProfileByEmail(username).then(p => p.id);
        const canComplete = await permissionService.canCompleteStep(userId, stepInstanceId);
        if (!canComplete) {
            return res.status(403).json({ error: 'No permission' });
        }

        // ❌ RouteでWorkflowロジック
        const { createWorkflowEngine, CompleteStepCommand } = require('../workflow/core/WorkflowEngineV3');
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(
            stepInstanceId,
            userId,
            responses,
            additionalContext
        );
        const events = await engine.executeCommand(command);

        // ❌ RouteでImpersonation処理
        if (res.locals.isImpersonating) {
            impersonationContextStore.storeContext(stepInstanceId, {
                originalUserId: res.locals.originalUserId,
                effectiveUserId: userId,
            });
        }

        // ❌ Routeでレスポンス処理
        const post = await PrismaService.main.post.findUnique({
            where: { id: postData.id },
            include: { comments: true },
        });

        // ❌ Routeで権限確認
        await checkPostPermissions(post, userId);

        // ... 100行以上のビジネスロジック

        res.json({ success: true, data: result });
    } catch (e) {
        handler.handleException(res, e);
    }
});
```

**なぜ酷いのか:**
- 200行以上のビジネスロジック
- テストしにくい（HTTPモッキングが必要）
- 再利用しにくい（routeに依存）
- 混在した責任
- デバッグしにくい
- パフォーマンス追跡しにくい

### リファクタリング方法（ステップバイステップ）

**ステップ1: Controller作成**

```typescript
// controllers/PostController.ts
export class PostController extends BaseController {
    private postService: PostService;

    constructor() {
        super();
        this.postService = new PostService();
    }

    async createPost(req: Request, res: Response): Promise<void> {
        try {
            const validated = createPostSchema.parse({
                ...req.body,
            });

            const result = await this.postService.createPost(
                validated,
                res.locals.userId
            );

            this.handleSuccess(res, result, 'Post created successfully');
        } catch (error) {
            this.handleError(error, res, 'createPost');
        }
    }
}
```

**ステップ2: Service作成**

```typescript
// services/postService.ts
export class PostService {
    async createPost(
        data: CreatePostDTO,
        userId: string
    ): Promise<PostResult> {
        // 権限確認
        const canCreate = await permissionService.canCreatePost(userId);
        if (!canCreate) {
            throw new ForbiddenError('No permission to create post');
        }

        // Workflow実行
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(/* ... */);
        const events = await engine.executeCommand(command);

        // 必要に応じてimpersonation処理
        if (context.isImpersonating) {
            await this.handleImpersonation(data.stepInstanceId, context);
        }

        // ロール同期
        await this.synchronizeRoles(events, userId);

        return { events, success: true };
    }

    private async handleImpersonation(stepInstanceId: number, context: any) {
        impersonationContextStore.storeContext(stepInstanceId, {
            originalUserId: context.originalUserId,
            effectiveUserId: context.effectiveUserId,
        });
    }

    private async synchronizeRoles(events: WorkflowEvent[], userId: string) {
        // ロール同期ロジック
    }
}
```

**ステップ3: Route更新**

```typescript
// routes/postRoutes.ts
import { PostController } from '../controllers/PostController';

const router = Router();
const controller = new PostController();

// ✅ クリーン: ルーティングのみ
router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createPost(req, res)
);
```

**結果:**
- Route: 8行（200行以上だった）
- Controller: 25行（リクエスト処理）
- Service: 50行（ビジネスロジック）
- テスト可能、再利用可能、保守可能！

---

## エラー処理

### Controllerエラー処理

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.userService.create(req.body);
        this.handleSuccess(res, result, 'User created', 201);
    } catch (error) {
        // BaseController.handleErrorが自動的に:
        // - コンテキストとともにSentryにキャプチャ
        // - 適切なステータスコード設定
        // - フォーマットされたエラーレスポンス返却
        this.handleError(error, res, 'createUser');
    }
}
```

### カスタムエラーステータスコード

```typescript
async getUser(req: Request, res: Response): Promise<void> {
    try {
        const user = await this.userService.findById(req.params.id);

        if (!user) {
            // カスタム404ステータス
            return this.handleError(
                new Error('User not found'),
                res,
                'getUser',
                404  // カスタムステータスコード
            );
        }

        this.handleSuccess(res, user);
    } catch (error) {
        this.handleError(error, res, 'getUser');
    }
}
```

### バリデーションエラー

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const validated = createUserSchema.parse(req.body);
        const user = await this.userService.create(validated);
        this.handleSuccess(res, user, 'User created', 201);
    } catch (error) {
        // Zodエラーは400ステータス
        if (error instanceof z.ZodError) {
            return this.handleError(error, res, 'createUser', 400);
        }
        this.handleError(error, res, 'createUser');
    }
}
```

---

## HTTPステータスコード

### 標準コード

| コード | ユースケース | 例 |
|------|----------|---------|
| 200 | 成功（GET、PUT） | ユーザー取得、更新 |
| 201 | 作成完了（POST） | ユーザー作成 |
| 204 | 内容なし（DELETE） | ユーザー削除 |
| 400 | 不正なリクエスト | 無効な入力データ |
| 401 | 認証なし | 認証されていない |
| 403 | 禁止 | 権限なし |
| 404 | 見つからない | リソースが存在しない |
| 409 | 競合 | 重複リソース |
| 422 | 処理できないエンティティ | バリデーション失敗 |
| 500 | 内部サーバーエラー | 予期しないエラー |

### 使用例

```typescript
// 200 - 成功（デフォルト）
this.handleSuccess(res, user);

// 201 - 作成完了
this.handleSuccess(res, user, 'Created', 201);

// 400 - 不正なリクエスト
this.handleError(error, res, 'operation', 400);

// 404 - 見つからない
this.handleError(new Error('Not found'), res, 'operation', 404);

// 403 - 禁止
this.handleError(new ForbiddenError('No permission'), res, 'operation', 403);
```

---

## リファクタリングガイド

### リファクタリングが必要なRoutesの識別

**危険信号:**
- Routeファイル > 100行
- 1つのrouteに複数のtry-catchブロック
- 直接データベースアクセス（Prisma呼び出し）
- 複雑なビジネスロジック（if文、ループ）
- Routesでの権限確認

**routes確認:**
```bash
# 大きなrouteファイルを見つける
wc -l form/src/routes/*.ts | sort -n

# Prisma使用があるroutesを見つける
grep -r "PrismaService" form/src/routes/
```

### リファクタリングプロセス

**1. Controllerに抽出:**
```typescript
// 以前: ロジックがあるRoute
router.post('/action', async (req, res) => {
    try {
        // 50行のロジック
    } catch (e) {
        handler.handleException(res, e);
    }
});

// 以後: クリーンなroute
router.post('/action', (req, res) => controller.performAction(req, res));

// 新しいcontrollerメソッド
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.service.performAction(req.body);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}
```

**2. Serviceに抽出:**
```typescript
// Controllerは薄く保つ
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const validated = actionSchema.parse(req.body);
        const result = await this.actionService.execute(validated);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}

// Serviceがビジネスロジックを含む
export class ActionService {
    async execute(data: ActionDTO): Promise<Result> {
        // すべてのビジネスロジックはここ
        // 権限確認
        // データベース操作
        // 複雑な変換
        return result;
    }
}
```

**3. Repository追加（必要な場合）:**
```typescript
// Serviceがrepositoryを呼び出す
export class ActionService {
    constructor(private actionRepository: ActionRepository) {}

    async execute(data: ActionDTO): Promise<Result> {
        // ビジネスロジック
        const entity = await this.actionRepository.findById(data.id);
        // さらなるロジック
        return await this.actionRepository.update(data.id, changes);
    }
}

// Repositoryがデータアクセスを処理
export class ActionRepository {
    async findById(id: number): Promise<Entity | null> {
        return PrismaService.main.entity.findUnique({ where: { id } });
    }

    async update(id: number, data: Partial<Entity>): Promise<Entity> {
        return PrismaService.main.entity.update({ where: { id }, data });
    }
}
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインガイド
- [services-and-repositories.md](services-and-repositories.md) - Serviceレイヤーの詳細
- [complete-examples.md](complete-examples.md) - 完全なリファクタリング例
