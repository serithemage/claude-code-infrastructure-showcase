---
name: backend-dev-guidelines
description: Node.js/Express/TypeScript 마이크로서비스를 위한 종합 백엔드 개발 가이드. routes, controllers, services, repositories, middleware 생성 시 또는 Express API, Prisma 데이터베이스 액세스, Sentry error tracking, Zod 검증, unifiedConfig, dependency injection, async 패턴 작업 시 사용. Layered architecture (routes → controllers → services → repositories), BaseController 패턴, error handling, performance monitoring, 테스트 전략, 레거시 패턴 마이그레이션을 다룹니다.
---

# 백엔드 개발 가이드라인

## 목적

최신 Node.js/Express/TypeScript 패턴을 사용하여 백엔드 마이크로서비스(blog-api, auth-service, notifications-service) 전반에 일관성과 모범 사례를 확립합니다.

## 이 Skill 사용 시점

다음 작업 시 자동 활성화됩니다:
- Routes, endpoints, APIs 생성 또는 수정
- Controllers, services, repositories 구축
- Middleware 구현 (auth, validation, error handling)
- Prisma를 사용한 데이터베이스 작업
- Sentry를 사용한 error tracking
- Zod를 사용한 입력 검증
- 설정 관리
- 백엔드 테스팅 및 리팩토링

---

## 빠른 시작

### 새 백엔드 기능 체크리스트

- [ ] **Route**: 깔끔한 정의, controller에 위임
- [ ] **Controller**: BaseController 상속
- [ ] **Service**: DI를 사용한 비즈니스 로직
- [ ] **Repository**: 데이터베이스 액세스 (복잡한 경우)
- [ ] **Validation**: Zod 스키마
- [ ] **Sentry**: Error tracking
- [ ] **Tests**: Unit + integration 테스트
- [ ] **Config**: unifiedConfig 사용

### 새 마이크로서비스 체크리스트

- [ ] 디렉토리 구조 ([architecture-overview.md](architecture-overview.md) 참조)
- [ ] Sentry용 instrument.ts
- [ ] unifiedConfig 설정
- [ ] BaseController 클래스
- [ ] Middleware 스택
- [ ] Error boundary
- [ ] 테스팅 프레임워크

---

## 아키텍처 개요

### Layered Architecture

```
HTTP Request
    ↓
Routes (routing만)
    ↓
Controllers (요청 처리)
    ↓
Services (비즈니스 로직)
    ↓
Repositories (데이터 액세스)
    ↓
Database (Prisma)
```

**핵심 원칙:** 각 레이어는 하나의 책임만 가집니다.

자세한 내용은 [architecture-overview.md](architecture-overview.md)를 참조하세요.

---

## 디렉토리 구조

```
service/src/
├── config/              # UnifiedConfig
├── controllers/         # 요청 핸들러
├── services/            # 비즈니스 로직
├── repositories/        # 데이터 액세스
├── routes/              # Route 정의
├── middleware/          # Express middleware
├── types/               # TypeScript 타입
├── validators/          # Zod 스키마
├── utils/               # 유틸리티
├── tests/               # 테스트
├── instrument.ts        # Sentry (첫 번째 IMPORT)
├── app.ts               # Express 설정
└── server.ts            # HTTP 서버
```

**명명 규칙:**
- Controllers: `PascalCase` - `UserController.ts`
- Services: `camelCase` - `userService.ts`
- Routes: `camelCase + Routes` - `userRoutes.ts`
- Repositories: `PascalCase + Repository` - `UserRepository.ts`

---

## 핵심 원칙 (7가지 핵심 규칙)

### 1. Routes는 라우팅만, Controllers가 제어

```typescript
// ❌ 절대 안 됨: routes에 비즈니스 로직
router.post('/submit', async (req, res) => {
    // 200줄의 로직
});

// ✅ 항상: controller에 위임
router.post('/submit', (req, res) => controller.submit(req, res));
```

### 2. 모든 Controllers는 BaseController 상속

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

### 3. 모든 에러는 Sentry로

```typescript
try {
    await operation();
} catch (error) {
    Sentry.captureException(error);
    throw error;
}
```

### 4. unifiedConfig 사용, process.env 절대 사용 금지

```typescript
// ❌ 절대 안 됨
const timeout = process.env.TIMEOUT_MS;

// ✅ 항상
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
```

### 5. 모든 입력은 Zod로 검증

```typescript
const schema = z.object({ email: z.string().email() });
const validated = schema.parse(req.body);
```

### 6. 데이터 액세스에 Repository 패턴 사용

```typescript
// Service → Repository → Database
const users = await userRepository.findActive();
```

### 7. 종합 테스팅 필수

```typescript
describe('UserService', () => {
    it('should create user', async () => {
        expect(user).toBeDefined();
    });
});
```

---

## 공통 Imports

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

## 빠른 참조

### HTTP 상태 코드

| 코드 | 사용 사례 |
|------|----------|
| 200 | 성공 |
| 201 | 생성됨 |
| 400 | 잘못된 요청 |
| 401 | 인증 안 됨 |
| 403 | 금지됨 |
| 404 | 찾을 수 없음 |
| 500 | 서버 오류 |

### 서비스 템플릿

**Blog API** (✅ 성숙) - REST API 템플릿으로 사용
**Auth Service** (✅ 성숙) - 인증 패턴 템플릿으로 사용

---

## 피해야 할 안티패턴

❌ Routes에 비즈니스 로직
❌ process.env 직접 사용
❌ Error handling 누락
❌ 입력 검증 없음
❌ 모든 곳에서 Prisma 직접 사용
❌ Sentry 대신 console.log

---

## 네비게이션 가이드

| 필요한 작업... | 읽어야 할 문서 |
|------------|-----------|
| 아키텍처 이해 | [architecture-overview.md](architecture-overview.md) |
| Routes/controllers 생성 | [routing-and-controllers.md](routing-and-controllers.md) |
| 비즈니스 로직 구성 | [services-and-repositories.md](services-and-repositories.md) |
| 입력 검증 | [validation-patterns.md](validation-patterns.md) |
| Error tracking 추가 | [sentry-and-monitoring.md](sentry-and-monitoring.md) |
| Middleware 생성 | [middleware-guide.md](middleware-guide.md) |
| 데이터베이스 액세스 | [database-patterns.md](database-patterns.md) |
| 설정 관리 | [configuration.md](configuration.md) |
| Async/errors 처리 | [async-and-errors.md](async-and-errors.md) |
| 테스트 작성 | [testing-guide.md](testing-guide.md) |
| 예시 보기 | [complete-examples.md](complete-examples.md) |

---

## 리소스 파일

### [architecture-overview.md](architecture-overview.md)
Layered architecture, 요청 수명 주기, 관심사 분리

### [routing-and-controllers.md](routing-and-controllers.md)
Route 정의, BaseController, error handling, 예시

### [services-and-repositories.md](services-and-repositories.md)
Service 패턴, DI, repository 패턴, 캐싱

### [validation-patterns.md](validation-patterns.md)
Zod 스키마, 검증, DTO 패턴

### [sentry-and-monitoring.md](sentry-and-monitoring.md)
Sentry 초기화, 오류 캡처, performance monitoring

### [middleware-guide.md](middleware-guide.md)
Auth, audit, error boundaries, AsyncLocalStorage

### [database-patterns.md](database-patterns.md)
PrismaService, repositories, 트랜잭션, 최적화

### [configuration.md](configuration.md)
UnifiedConfig, 환경 설정, 시크릿

### [async-and-errors.md](async-and-errors.md)
Async 패턴, 커스텀 에러, asyncErrorWrapper

### [testing-guide.md](testing-guide.md)
Unit/integration 테스트, mocking, 커버리지

### [complete-examples.md](complete-examples.md)
전체 예시, 리팩토링 가이드

---

## 관련 Skills

- **database-verification** - 컬럼 이름 및 스키마 일관성 검증
- **error-tracking** - Sentry 통합 패턴
- **skill-developer** - Skills 생성 및 관리를 위한 메타 skill

---

**Skill 상태**: 완료 ✅
**줄 수**: < 500 ✅
**Progressive Disclosure**: 11개 리소스 파일 ✅
