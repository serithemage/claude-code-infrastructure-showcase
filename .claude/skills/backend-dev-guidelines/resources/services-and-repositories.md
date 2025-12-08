# ServicesとRepositories - ビジネスロジック層

Servicesでビジネスロジックを構成し、repositoriesでデータアクセスを管理する完全なガイドです。

## 目次

- [Service層概要](#service層概要)
- [依存性注入パターン](#依存性注入パターン)
- [Singletonパターン](#singletonパターン)
- [Repositoryパターン](#repositoryパターン)
- [Service設計原則](#service設計原則)
- [キャッシング戦略](#キャッシング戦略)
- [Servicesのテスト](#servicesのテスト)

---

## Service層概要

### Servicesの目的

**Servicesはビジネスロジックを含む** - アプリケーションの'何を'と'なぜ':

```
Controllerの質問: "これを行うべきですか？"
Serviceの回答: "はい/いいえ、理由はこれで、これが発生します"
Repositoryの実行: "要求されたデータです"
```

**Servicesの責任:**
- ✅ ビジネスルール適用
- ✅ 複数のrepositoriesのオーケストレーション
- ✅ トランザクション管理
- ✅ 複雑な計算
- ✅ 外部サービス統合
- ✅ ビジネスバリデーション

**Servicesがすべきでないこと:**
- ❌ HTTPを知ること（Request/Response）
- ❌ Prisma直接アクセス（repositories使用）
- ❌ Route専用ロジック処理
- ❌ HTTPレスポンスフォーマット

---

## 依存性注入パターン

### 依存性注入を使用する理由

**利点:**
- テストしやすい（mock注入）
- 明確な依存関係
- 柔軟な設定
- 疎結合を促進

### 優れた例: NotificationService

**ファイル:** `/blog-api/src/services/NotificationService.ts`

```typescript
// 明確さのための依存性インターフェース定義
export interface NotificationServiceDependencies {
    prisma: PrismaClient;
    batchingService: BatchingService;
    emailComposer: EmailComposer;
}

// 依存性注入があるService
export class NotificationService {
    private prisma: PrismaClient;
    private batchingService: BatchingService;
    private emailComposer: EmailComposer;
    private preferencesCache: Map<string, { preferences: UserPreference; timestamp: number }> = new Map();
    private CACHE_TTL = (notificationConfig.preferenceCacheTTLMinutes || 5) * 60 * 1000;

    // コンストラクタを通じて依存性注入
    constructor(dependencies: NotificationServiceDependencies) {
        this.prisma = dependencies.prisma;
        this.batchingService = dependencies.batchingService;
        this.emailComposer = dependencies.emailComposer;
    }

    /**
     * 通知を作成し適切にルーティング
     */
    async createNotification(params: CreateNotificationParams) {
        const { recipientID, type, title, message, link, context = {}, channel = 'both', priority = NotificationPriority.NORMAL } = params;

        try {
            // テンプレートを取得してコンテンツをレンダリング
            const template = getNotificationTemplate(type);
            const rendered = renderNotificationContent(template, context);

            // インアプリ通知レコードを作成
            const notificationId = await createNotificationRecord({
                instanceId: parseInt(context.instanceId || '0', 10),
                template: type,
                recipientUserId: recipientID,
                channel: channel === 'email' ? 'email' : 'inApp',
                contextData: context,
                title: finalTitle,
                message: finalMessage,
                link: finalLink,
            });

            // チャンネルに応じて通知をルーティング
            if (channel === 'email' || channel === 'both') {
                await this.routeNotification({
                    notificationId,
                    userId: recipientID,
                    type,
                    priority,
                    title: finalTitle,
                    message: finalMessage,
                    link: finalLink,
                    context,
                });
            }

            return notification;
        } catch (error) {
            ErrorLogger.log(error, {
                context: {
                    '[NotificationService] createNotification': {
                        type: params.type,
                        recipientID: params.recipientID,
                    },
                },
            });
            throw error;
        }
    }

    /**
     * ユーザー設定に基づいて通知をルーティング
     */
    private async routeNotification(params: { notificationId: number; userId: string; type: string; priority: NotificationPriority; title: string; message: string; link?: string; context?: Record<string, any> }) {
        // キャッシングとともにユーザー設定を取得
        const preferences = await this.getUserPreferences(params.userId);

        // バッチするか即時送信するか確認
        if (this.shouldBatchEmail(preferences, params.type, params.priority)) {
            await this.batchingService.queueNotificationForBatch({
                notificationId: params.notificationId,
                userId: params.userId,
                userPreference: preferences,
                priority: params.priority,
            });
        } else {
            // EmailComposerを通じて即時送信
            await this.sendImmediateEmail({
                userId: params.userId,
                title: params.title,
                message: params.message,
                link: params.link,
                context: params.context,
                type: params.type,
            });
        }
    }

    /**
     * メールをバッチすべきかを決定
     */
    shouldBatchEmail(preferences: UserPreference, notificationType: string, priority: NotificationPriority): boolean {
        // HIGH優先度は常に即時
        if (priority === NotificationPriority.HIGH) {
            return false;
        }

        // バッチモードを確認
        const batchMode = preferences.emailBatchMode || BatchMode.IMMEDIATE;
        return batchMode !== BatchMode.IMMEDIATE;
    }

    /**
     * キャッシングとともにユーザー設定を取得
     */
    async getUserPreferences(userId: string): Promise<UserPreference> {
        // まずキャッシュを確認
        const cached = this.preferencesCache.get(userId);
        if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
            return cached.preferences;
        }

        const preference = await this.prisma.userPreference.findUnique({
            where: { userID: userId },
        });

        const finalPreferences = preference || DEFAULT_PREFERENCES;

        // キャッシュを更新
        this.preferencesCache.set(userId, {
            preferences: finalPreferences,
            timestamp: Date.now(),
        });

        return finalPreferences;
    }
}
```

**Controllerでの使用:**

```typescript
// 依存性とともにインスタンス化
const notificationService = new NotificationService({
    prisma: PrismaService.main,
    batchingService: new BatchingService(PrismaService.main),
    emailComposer: new EmailComposer(),
});

// Controllerで使用
const notification = await notificationService.createNotification({
    recipientID: 'user-123',
    type: 'AFRLWorkflowNotification',
    context: { workflowName: 'AFRL Monthly Report' },
});
```

**キーポイント:**
- コンストラクタを通じて依存性を渡す
- 明確なインターフェースが必要な依存性を定義
- テストしやすい（mock注入）
- カプセル化されたキャッシングロジック
- HTTPから分離されたビジネスルール

---

## Singletonパターン

### Singletonを使用すべき場合

**使用対象:**
- 高コストな初期化があるServices
- 共有状態があるServices（キャッシング）
- 複数の場所からアクセスされるServices
- Permission services
- Configuration services

### 例: PermissionService（Singleton）

**ファイル:** `/blog-api/src/services/permissionService.ts`

```typescript
import { PrismaClient } from '@prisma/client';

class PermissionService {
    private static instance: PermissionService;
    private prisma: PrismaClient;
    private permissionCache: Map<string, { canAccess: boolean; timestamp: number }> = new Map();
    private CACHE_TTL = 5 * 60 * 1000; // 5分

    // privateコンストラクタで直接インスタンス化を防止
    private constructor() {
        this.prisma = PrismaService.main;
    }

    // Singletonインスタンスを取得
    public static getInstance(): PermissionService {
        if (!PermissionService.instance) {
            PermissionService.instance = new PermissionService();
        }
        return PermissionService.instance;
    }

    /**
     * ユーザーがworkflowステップを完了できるか確認
     */
    async canCompleteStep(userId: string, stepInstanceId: number): Promise<boolean> {
        const cacheKey = `${userId}:${stepInstanceId}`;

        // キャッシュ確認
        const cached = this.permissionCache.get(cacheKey);
        if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
            return cached.canAccess;
        }

        try {
            const post = await this.prisma.post.findUnique({
                where: { id: postId },
                include: {
                    author: true,
                    comments: {
                        include: {
                            user: true,
                        },
                    },
                },
            });

            if (!post) {
                return false;
            }

            // ユーザーに権限があるか確認
            const canEdit = post.authorId === userId ||
                await this.isUserAdmin(userId);

            // 結果をキャッシュ
            this.permissionCache.set(cacheKey, {
                canAccess: isAssigned,
                timestamp: Date.now(),
            });

            return isAssigned;
        } catch (error) {
            console.error('[PermissionService] Error checking step permission:', error);
            return false;
        }
    }

    /**
     * ユーザーキャッシュをクリア
     */
    clearUserCache(userId: string): void {
        for (const [key] of this.permissionCache) {
            if (key.startsWith(`${userId}:`)) {
                this.permissionCache.delete(key);
            }
        }
    }

    /**
     * 全キャッシュをクリア
     */
    clearCache(): void {
        this.permissionCache.clear();
    }
}

// Singletonインスタンスをexport
export const permissionService = PermissionService.getInstance();
```

**使用:**

```typescript
import { permissionService } from '../services/permissionService';

// コードベースのどこからでも使用
const canComplete = await permissionService.canCompleteStep(userId, stepId);

if (!canComplete) {
    throw new ForbiddenError('You do not have permission to complete this step');
}
```

---

## Repositoryパターン

### Repositoriesの目的

**Repositoriesはデータアクセスを抽象化** - データ操作の'どのように':

```
Service: "名前順にソートされたすべてのアクティブユーザーをください"
Repository: "これを実行するPrismaクエリです"
```

**Repositoriesの責任:**
- ✅ すべてのPrisma操作
- ✅ クエリ構築
- ✅ クエリ最適化（select、include）
- ✅ データベースエラー処理
- ✅ データベース結果のキャッシング

**Repositoriesがすべきでないこと:**
- ❌ ビジネスロジックの含有
- ❌ HTTPを知ること
- ❌ 決定を下すこと（それはservice層）

### Repositoryテンプレート

```typescript
// repositories/UserRepository.ts
import { PrismaService } from '@project-lifecycle-portal/database';
import type { User, Prisma } from '@project-lifecycle-portal/database';

export class UserRepository {
    /**
     * 最適化されたクエリでIDでユーザーを見つける
     */
    async findById(userId: string): Promise<User | null> {
        try {
            return await PrismaService.main.user.findUnique({
                where: { userID: userId },
                select: {
                    userID: true,
                    email: true,
                    name: true,
                    isActive: true,
                    roles: true,
                    createdAt: true,
                    updatedAt: true,
                },
            });
        } catch (error) {
            console.error('[UserRepository] Error finding user by ID:', error);
            throw new Error(`Failed to find user: ${userId}`);
        }
    }

    /**
     * すべてのアクティブユーザーを見つける
     */
    async findActive(options?: { orderBy?: Prisma.UserOrderByWithRelationInput }): Promise<User[]> {
        try {
            return await PrismaService.main.user.findMany({
                where: { isActive: true },
                orderBy: options?.orderBy || { name: 'asc' },
                select: {
                    userID: true,
                    email: true,
                    name: true,
                    roles: true,
                },
            });
        } catch (error) {
            console.error('[UserRepository] Error finding active users:', error);
            throw new Error('Failed to find active users');
        }
    }

    /**
     * メールでユーザーを見つける
     */
    async findByEmail(email: string): Promise<User | null> {
        try {
            return await PrismaService.main.user.findUnique({
                where: { email },
            });
        } catch (error) {
            console.error('[UserRepository] Error finding user by email:', error);
            throw new Error(`Failed to find user with email: ${email}`);
        }
    }

    /**
     * 新しいユーザーを作成
     */
    async create(data: Prisma.UserCreateInput): Promise<User> {
        try {
            return await PrismaService.main.user.create({ data });
        } catch (error) {
            console.error('[UserRepository] Error creating user:', error);
            throw new Error('Failed to create user');
        }
    }

    /**
     * ユーザーを更新
     */
    async update(userId: string, data: Prisma.UserUpdateInput): Promise<User> {
        try {
            return await PrismaService.main.user.update({
                where: { userID: userId },
                data,
            });
        } catch (error) {
            console.error('[UserRepository] Error updating user:', error);
            throw new Error(`Failed to update user: ${userId}`);
        }
    }

    /**
     * ユーザーを削除（isActive = falseでsoft delete）
     */
    async delete(userId: string): Promise<User> {
        try {
            return await PrismaService.main.user.update({
                where: { userID: userId },
                data: { isActive: false },
            });
        } catch (error) {
            console.error('[UserRepository] Error deleting user:', error);
            throw new Error(`Failed to delete user: ${userId}`);
        }
    }

    /**
     * メール存在確認
     */
    async emailExists(email: string): Promise<boolean> {
        try {
            const count = await PrismaService.main.user.count({
                where: { email },
            });
            return count > 0;
        } catch (error) {
            console.error('[UserRepository] Error checking email exists:', error);
            throw new Error('Failed to check if email exists');
        }
    }
}

// Singletonインスタンスをexport
export const userRepository = new UserRepository();
```

**ServiceでRepositoryを使用:**

```typescript
// services/userService.ts
import { userRepository } from '../repositories/UserRepository';
import { ConflictError, NotFoundError } from '../utils/errors';

export class UserService {
    /**
     * ビジネスルールとともに新しいユーザーを作成
     */
    async createUser(data: { email: string; name: string; roles: string[] }): Promise<User> {
        // ビジネスルール: メールが既に存在するか確認
        const emailExists = await userRepository.emailExists(data.email);
        if (emailExists) {
            throw new ConflictError('Email already exists');
        }

        // ビジネスルール: ロール検証
        const validRoles = ['admin', 'operations', 'user'];
        const invalidRoles = data.roles.filter((role) => !validRoles.includes(role));
        if (invalidRoles.length > 0) {
            throw new ValidationError(`Invalid roles: ${invalidRoles.join(', ')}`);
        }

        // Repositoryを通じてユーザー作成
        return await userRepository.create({
            email: data.email,
            name: data.name,
            roles: data.roles,
            isActive: true,
        });
    }

    /**
     * IDでユーザーを取得
     */
    async getUser(userId: string): Promise<User> {
        const user = await userRepository.findById(userId);

        if (!user) {
            throw new NotFoundError(`User not found: ${userId}`);
        }

        return user;
    }
}
```

---

## Service設計原則

### 1. 単一責任

各serviceは1つの明確な目的を持つべき:

```typescript
// ✅ 良い - 単一責任
class UserService {
    async createUser() {}
    async updateUser() {}
    async deleteUser() {}
}

class EmailService {
    async sendEmail() {}
    async sendBulkEmails() {}
}

// ❌ 悪い - 責任が多すぎる
class UserService {
    async createUser() {}
    async sendWelcomeEmail() {}  // EmailServiceであるべき
    async logUserActivity() {}   // AuditServiceであるべき
    async processPayment() {}    // PaymentServiceであるべき
}
```

### 2. 明確なメソッド名

メソッド名は何をするか説明すべき:

```typescript
// ✅ 良い - 明確な意図
async createNotification()
async getUserPreferences()
async shouldBatchEmail()
async routeNotification()

// ❌ 悪い - 曖昧または誤解を招く
async process()
async handle()
async doIt()
async execute()
```

### 3. 戻り値の型

常に明示的な戻り値の型を使用:

```typescript
// ✅ 良い - 明示的な型
async createUser(data: CreateUserDTO): Promise<User> {}
async findUsers(): Promise<User[]> {}
async deleteUser(id: string): Promise<void> {}

// ❌ 悪い - 暗黙のany
async createUser(data) {}  // 型なし！
```

### 4. エラー処理

Servicesは意味のあるエラーをスローすべき:

```typescript
// ✅ 良い - 意味のあるエラー
if (!user) {
    throw new NotFoundError(`User not found: ${userId}`);
}

if (emailExists) {
    throw new ConflictError('Email already exists');
}

// ❌ 悪い - 一般的なエラー
if (!user) {
    throw new Error('Error');  // 何のエラー？
}
```

### 5. God Servicesを避ける

すべてを行うservicesを作らない:

```typescript
// ❌ 悪い - God service
class WorkflowService {
    async startWorkflow() {}
    async completeStep() {}
    async assignRoles() {}
    async sendNotifications() {}  // NotificationServiceであるべき
    async validatePermissions() {}  // PermissionServiceであるべき
    async logAuditTrail() {}  // AuditServiceであるべき
    // ... 50以上のメソッド
}

// ✅ 良い - 集中したservices
class WorkflowService {
    constructor(
        private notificationService: NotificationService,
        private permissionService: PermissionService,
        private auditService: AuditService
    ) {}

    async startWorkflow() {
        // 他のservicesをオーケストレーション
        await this.permissionService.checkPermission();
        await this.workflowRepository.create();
        await this.notificationService.notify();
        await this.auditService.log();
    }
}
```

---

## キャッシング戦略

### 1. インメモリキャッシング

```typescript
class UserService {
    private cache: Map<string, { user: User; timestamp: number }> = new Map();
    private CACHE_TTL = 5 * 60 * 1000; // 5分

    async getUser(userId: string): Promise<User> {
        // キャッシュ確認
        const cached = this.cache.get(userId);
        if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
            return cached.user;
        }

        // データベースから取得
        const user = await userRepository.findById(userId);

        // キャッシュ更新
        if (user) {
            this.cache.set(userId, { user, timestamp: Date.now() });
        }

        return user;
    }

    clearUserCache(userId: string): void {
        this.cache.delete(userId);
    }
}
```

### 2. キャッシュ無効化

```typescript
class UserService {
    async updateUser(userId: string, data: UpdateUserDTO): Promise<User> {
        // データベースで更新
        const user = await userRepository.update(userId, data);

        // キャッシュ無効化
        this.clearUserCache(userId);

        return user;
    }
}
```

---

## Servicesのテスト

### 単体テスト

```typescript
// tests/userService.test.ts
import { UserService } from '../services/userService';
import { userRepository } from '../repositories/UserRepository';
import { ConflictError } from '../utils/errors';

// Repositoryをモック
jest.mock('../repositories/UserRepository');

describe('UserService', () => {
    let userService: UserService;

    beforeEach(() => {
        userService = new UserService();
        jest.clearAllMocks();
    });

    describe('createUser', () => {
        it('メールが存在しない場合ユーザーを作成すべき', async () => {
            // Arrange
            const userData = {
                email: 'test@example.com',
                name: 'Test User',
                roles: ['user'],
            };

            (userRepository.emailExists as jest.Mock).mockResolvedValue(false);
            (userRepository.create as jest.Mock).mockResolvedValue({
                userID: '123',
                ...userData,
            });

            // Act
            const user = await userService.createUser(userData);

            // Assert
            expect(user).toBeDefined();
            expect(user.email).toBe(userData.email);
            expect(userRepository.emailExists).toHaveBeenCalledWith(userData.email);
            expect(userRepository.create).toHaveBeenCalled();
        });

        it('メールが存在する場合ConflictErrorをスローすべき', async () => {
            // Arrange
            const userData = {
                email: 'existing@example.com',
                name: 'Test User',
                roles: ['user'],
            };

            (userRepository.emailExists as jest.Mock).mockResolvedValue(true);

            // Act & Assert
            await expect(userService.createUser(userData)).rejects.toThrow(ConflictError);
            expect(userRepository.create).not.toHaveBeenCalled();
        });
    });
});
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインガイド
- [routing-and-controllers.md](routing-and-controllers.md) - Servicesを使用するControllers
- [database-patterns.md](database-patterns.md) - Prismaとrepositoryパターン
- [complete-examples.md](complete-examples.md) - 完全なservice/repository例
