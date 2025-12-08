---
name: backend-dev-guidelines
description: Node.js/Express/TypeScriptマイクロサービスのための包括的なバックエンド開発ガイド。routes、controllers、services、repositories、middleware作成時、またはExpress API、Prismaデータベースアクセス、Sentry error tracking、Zod検証、unifiedConfig、dependency injection、asyncパターン作業時に使用。Layered architecture (routes → controllers → services → repositories)、BaseControllerパターン、error handling、performance monitoring、テスト戦略、レガシーパターンのマイグレーションをカバーします。
---

# バックエンド開発ガイドライン

## 目的

最新のNode.js/Express/TypeScriptパターンを使用して、バックエンドマイクロサービス(blog-api、auth-service、notifications-service)全般に一貫性とベストプラクティスを確立します。

## このSkill使用タイミング

以下の作業時に自動的に有効化されます：
- Routes、endpoints、APIsの作成または修正
- Controllers、services、repositoriesの構築
- Middleware実装（auth、validation、error handling）
- Prismaを使用したデータベース操作
- Sentryを使用したerror tracking
- Zodを使用した入力検証
- 設定管理
- バックエンドテストおよびリファクタリング

---

## クイックスタート

### 新規バックエンド機能チェックリスト

- [ ] **Route**: クリーンな定義、controllerに委譲
- [ ] **Controller**: BaseController継承
- [ ] **Service**: DIを使用したビジネスロジック
- [ ] **Repository**: データベースアクセス（複雑な場合）
- [ ] **Validation**: Zodスキーマ
- [ ] **Sentry**: Error tracking
- [ ] **Tests**: Unit + integration テスト
- [ ] **Config**: unifiedConfig使用

### 新規マイクロサービスチェックリスト

- [ ] ディレクトリ構造（[architecture-overview.md](architecture-overview.md)参照）
- [ ] Sentry用instrument.ts
- [ ] unifiedConfig設定
- [ ] BaseControllerクラス
- [ ] Middlewareスタック
- [ ] Error boundary
- [ ] テスティングフレームワーク

---

## アーキテクチャ概要

### Layered Architecture

```
HTTP Request
    ↓
Routes (routingのみ)
    ↓
Controllers (リクエスト処理)
    ↓
Services (ビジネスロジック)
    ↓
Repositories (データアクセス)
    ↓
Database (Prisma)
```

**核心原則:** 各レイヤーは単一の責任のみを持ちます。

詳細は[architecture-overview.md](architecture-overview.md)を参照してください。

---

## ディレクトリ構造

```
service/src/
├── config/              # UnifiedConfig
├── controllers/         # リクエストハンドラー
├── services/            # ビジネスロジック
├── repositories/        # データアクセス
├── routes/              # Route定義
├── middleware/          # Express middleware
├── types/               # TypeScript型
├── validators/          # Zodスキーマ
├── utils/               # ユーティリティ
├── tests/               # テスト
├── instrument.ts        # Sentry（最初のIMPORT）
├── app.ts               # Express設定
└── server.ts            # HTTPサーバー
```

**命名規則:**
- Controllers: `PascalCase` - `UserController.ts`
- Services: `camelCase` - `userService.ts`
- Routes: `camelCase + Routes` - `userRoutes.ts`
- Repositories: `PascalCase + Repository` - `UserRepository.ts`

---

## 核心原則（7つの核心ルール）

### 1. RoutesはRoutingのみ、ControllersがControl

```typescript
// ❌ 絶対NG: routesにビジネスロジック
router.post('/submit', async (req, res) => {
    // 200行のロジック
});

// ✅ 常に: controllerに委譲
router.post('/submit', (req, res) => controller.submit(req, res));
```

### 2. すべてのControllersはBaseController継承

```typescript
export class UserController extends BaseController {
    async getUser(req: Request, res: Response): Promise<void> {
        try {
            const user = await this.userService.findById(req.params.id);
            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }
}
```

### 3. すべてのエラーはSentryへ

```typescript
try {
    await operation();
} catch (error) {
    Sentry.captureException(error);
    throw error;
}
```

### 4. unifiedConfig使用、process.env絶対使用禁止

```typescript
// ❌ 絶対NG
const timeout = process.env.TIMEOUT_MS;

// ✅ 常に
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
```

### 5. すべての入力はZodで検証

```typescript
const schema = z.object({ email: z.string().email() });
const validated = schema.parse(req.body);
```

### 6. データアクセスにRepositoryパターン使用

```typescript
// Service → Repository → Database
const users = await userRepository.findActive();
```

### 7. 包括的テスティング必須

```typescript
describe('UserService', () => {
    it('should create user', async () => {
        expect(user).toBeDefined();
    });
});
```

---

## 共通Imports

```typescript
// Express
import express, { Request, Response, NextFunction, Router } from 'express';

// Validation
import { z } from 'zod';

// Database
import { PrismaClient } from '@prisma/client';
import type { Prisma } from '@prisma/client';

// Sentry
import * as Sentry from '@sentry/node';

// Config
import { config } from './config/unifiedConfig';

// Middleware
import { SSOMiddlewareClient } from './middleware/SSOMiddleware';
import { asyncErrorWrapper } from './middleware/errorBoundary';
```

---

## クイックリファレンス

### HTTPステータスコード

| コード | ユースケース |
|------|----------|
| 200 | 成功 |
| 201 | 作成完了 |
| 400 | 不正なリクエスト |
| 401 | 認証なし |
| 403 | 禁止 |
| 404 | 見つからない |
| 500 | サーバーエラー |

### サービステンプレート

**Blog API**（✅ 成熟）- REST APIテンプレートとして使用
**Auth Service**（✅ 成熟）- 認証パターンテンプレートとして使用

---

## 避けるべきアンチパターン

❌ Routesにビジネスロジック
❌ process.env直接使用
❌ Error handling漏れ
❌ 入力検証なし
❌ すべての場所でPrisma直接使用
❌ Sentry代わりにconsole.log

---

## ナビゲーションガイド

| 必要な作業... | 読むべきドキュメント |
|------------|-----------|
| アーキテクチャ理解 | [architecture-overview.md](architecture-overview.md) |
| Routes/controllers作成 | [routing-and-controllers.md](routing-and-controllers.md) |
| ビジネスロジック構成 | [services-and-repositories.md](services-and-repositories.md) |
| 入力検証 | [validation-patterns.md](validation-patterns.md) |
| Error tracking追加 | [sentry-and-monitoring.md](sentry-and-monitoring.md) |
| Middleware作成 | [middleware-guide.md](middleware-guide.md) |
| データベースアクセス | [database-patterns.md](database-patterns.md) |
| 設定管理 | [configuration.md](configuration.md) |
| Async/errors処理 | [async-and-errors.md](async-and-errors.md) |
| テスト作成 | [testing-guide.md](testing-guide.md) |
| 例示を見る | [complete-examples.md](complete-examples.md) |

---

## リソースファイル

### [architecture-overview.md](architecture-overview.md)
Layered architecture、リクエストライフサイクル、関心の分離

### [routing-and-controllers.md](routing-and-controllers.md)
Route定義、BaseController、error handling、例示

### [services-and-repositories.md](services-and-repositories.md)
Serviceパターン、DI、repositoryパターン、キャッシング

### [validation-patterns.md](validation-patterns.md)
Zodスキーマ、検証、DTOパターン

### [sentry-and-monitoring.md](sentry-and-monitoring.md)
Sentry初期化、エラーキャプチャ、performance monitoring

### [middleware-guide.md](middleware-guide.md)
Auth、audit、error boundaries、AsyncLocalStorage

### [database-patterns.md](database-patterns.md)
PrismaService、repositories、トランザクション、最適化

### [configuration.md](configuration.md)
UnifiedConfig、環境設定、シークレット

### [async-and-errors.md](async-and-errors.md)
Asyncパターン、カスタムエラー、asyncErrorWrapper

### [testing-guide.md](testing-guide.md)
Unit/integrationテスト、mocking、カバレッジ

### [complete-examples.md](complete-examples.md)
完全な例示、リファクタリングガイド

---

## 関連Skills

- **database-verification** - カラム名およびスキーマ一貫性検証
- **error-tracking** - Sentry統合パターン
- **skill-developer** - Skills作成および管理のためのメタskill

---

**Skillステータス**: 完了 ✅
**行数**: < 500 ✅
**Progressive Disclosure**: 11個のリソースファイル ✅
