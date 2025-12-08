# テストガイド - バックエンドテスト戦略

Jestとモベストプラクティスを使用したバックエンドサービステストの完全なガイドです。

## 目次

- [単体テスト](#単体テスト)
- [統合テスト](#統合テスト)
- [モッキング戦略](#モッキング戦略)
- [テストデータ管理](#テストデータ管理)
- [認証されたルートテスト](#認証されたルートテスト)
- [カバレッジ目標](#カバレッジ目標)

---

## 単体テスト

### テスト構造

```typescript
// services/userService.test.ts
import { UserService } from './userService';
import { UserRepository } from '../repositories/UserRepository';

jest.mock('../repositories/UserRepository');

describe('UserService', () => {
    let service: UserService;
    let mockRepository: jest.Mocked<UserRepository>;

    beforeEach(() => {
        mockRepository = {
            findByEmail: jest.fn(),
            create: jest.fn(),
        } as any;

        service = new UserService();
        (service as any).userRepository = mockRepository;
    });

    afterEach(() => {
        jest.clearAllMocks();
    });

    describe('create', () => {
        it('メールが存在する場合エラーをスローすべき', async () => {
            mockRepository.findByEmail.mockResolvedValue({ id: '123' } as any);

            await expect(
                service.create({ email: 'test@test.com' })
            ).rejects.toThrow('Email already in use');
        });

        it('メールがユニークな場合ユーザーを作成すべき', async () => {
            mockRepository.findByEmail.mockResolvedValue(null);
            mockRepository.create.mockResolvedValue({ id: '123' } as any);

            const user = await service.create({
                email: 'test@test.com',
                firstName: 'John',
                lastName: 'Doe',
            });

            expect(user).toBeDefined();
            expect(mockRepository.create).toHaveBeenCalledWith(
                expect.objectContaining({
                    email: 'test@test.com'
                })
            );
        });
    });
});
```

---

## 統合テスト

### 実際のデータベースでテスト

```typescript
import { PrismaService } from '@project-lifecycle-portal/database';

describe('UserService Integration', () => {
    let testUser: any;

    beforeAll(async () => {
        // テストデータ作成
        testUser = await PrismaService.main.user.create({
            data: {
                email: 'test@test.com',
                profile: { create: { firstName: 'Test', lastName: 'User' } },
            },
        });
    });

    afterAll(async () => {
        // クリーンアップ
        await PrismaService.main.user.delete({ where: { id: testUser.id } });
    });

    it('メールでユーザーを見つけるべき', async () => {
        const user = await userService.findByEmail('test@test.com');
        expect(user).toBeDefined();
        expect(user?.email).toBe('test@test.com');
    });
});
```

---

## モッキング戦略

### PrismaServiceモッキング

```typescript
jest.mock('@project-lifecycle-portal/database', () => ({
    PrismaService: {
        main: {
            user: {
                findMany: jest.fn(),
                findUnique: jest.fn(),
                create: jest.fn(),
                update: jest.fn(),
            },
        },
        isAvailable: true,
    },
}));
```

### Servicesモッキング

```typescript
const mockUserService = {
    findById: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
} as jest.Mocked<UserService>;
```

---

## テストデータ管理

### SetupとTeardown

```typescript
describe('PermissionService', () => {
    let instanceId: number;

    beforeAll(async () => {
        // テストpost作成
        const post = await PrismaService.main.post.create({
            data: { title: 'Test Post', content: 'Test', authorId: 'test-user' },
        });
        instanceId = post.id;
    });

    afterAll(async () => {
        // クリーンアップ
        await PrismaService.main.post.delete({
            where: { id: instanceId },
        });
    });

    beforeEach(() => {
        // キャッシュクリア
        permissionService.clearCache();
    });

    it('権限を確認すべき', async () => {
        const hasPermission = await permissionService.checkPermission(
            'user-id',
            instanceId,
            'VIEW_WORKFLOW'
        );
        expect(hasPermission).toBeDefined();
    });
});
```

---

## 認証されたルートテスト

### test-auth-route.jsを使用

```bash
# 認証されたエンドポイントをテスト
node scripts/test-auth-route.js http://localhost:3002/form/api/users

# POSTデータでテスト
node scripts/test-auth-route.js http://localhost:3002/form/api/users POST '{"email":"test@test.com"}'
```

### テストで認証をモック

```typescript
// auth middlewareをモック
jest.mock('../middleware/SSOMiddleware', () => ({
    SSOMiddlewareClient: {
        verifyLoginStatus: (req, res, next) => {
            res.locals.claims = {
                sub: 'test-user-id',
                preferred_username: 'testuser',
            };
            next();
        },
    },
}));
```

---

## カバレッジ目標

### 推奨カバレッジ

- **単体テスト**: 70%以上カバレッジ
- **統合テスト**: クリティカルパスをカバー
- **E2Eテスト**: 成功パスをカバー

### カバレッジ実行

```bash
npm test -- --coverage
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md)
- [services-and-repositories.md](services-and-repositories.md)
- [complete-examples.md](complete-examples.md)
