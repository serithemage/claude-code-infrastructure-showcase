---
name: error-tracking
description: 프로젝트 서비스에 Sentry v8 error tracking 및 performance monitoring 추가. 이 skill은 error handling 추가, 새 controllers 생성, cron jobs 계측, 데이터베이스 성능 추적 시 사용합니다. 모든 에러는 반드시 Sentry에 캡처되어야 합니다 - 예외 없음.
---

# 프로젝트 Sentry 통합 Skill

## 목적
이 skill은 Sentry v8 패턴을 따라 모든 프로젝트 서비스에서 종합적인 Sentry error tracking 및 performance monitoring을 강제합니다.

## 이 Skill 사용 시점
- 어떤 코드에든 error handling 추가
- 새 controllers 또는 routes 생성
- Cron jobs 계측
- 데이터베이스 성능 추적
- Performance spans 추가
- Workflow errors 처리

## 🚨 핵심 규칙

**모든 에러는 반드시 Sentry에 캡처되어야 합니다** - 예외 없음. console.error만 단독으로 사용하지 마세요.

## 현재 상태

### Form Service ✅ 완료
- Sentry v8 완전 통합
- 모든 workflow errors 추적됨
- SystemActionQueueProcessor 계측됨
- 테스트 엔드포인트 사용 가능

### Email Service 🟡 진행 중
- Phase 1-2 완료 (6/22 작업)
- 189개 ErrorLogger.log() 호출 남음

## Sentry 통합 패턴

### 1. Controller Error Handling

```typescript
// ✅ 올바름 - BaseController 사용
import { BaseController } from '../controllers/BaseController';

export class MyController extends BaseController {
    async myMethod() {
        try {
            // ... 코드
        } catch (error) {
            this.handleError(error, 'myMethod'); // 자동으로 Sentry에 전송
        }
    }
}
```

### 2. Route Error Handling (BaseController 없이)

```typescript
import * as Sentry from '@sentry/node';

router.get('/route', async (req, res) => {
    try {
        // ... 코드
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

// ✅ 올바름 - WorkflowSentryHelper 사용
WorkflowSentryHelper.captureWorkflowError(error, {
    workflowCode: 'DHS_CLOSEOUT',
    instanceId: 123,
    stepId: 456,
    userId: 'user-123',
    operation: 'stepCompletion',
    metadata: { additionalInfo: 'value' }
});
```

### 4. Cron Jobs (필수 패턴)

```typescript
#!/usr/bin/env node
// shebang 다음 첫 번째 줄 - 중요!
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
            // cron job 로직
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
        console.log('[Job] 성공적으로 완료');
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

// ✅ 올바름 - 데이터베이스 작업 래핑
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

### 6. Spans를 사용한 Async 작업

```typescript
import * as Sentry from '@sentry/node';

const result = await Sentry.startSpan({
    name: 'operation.name',
    op: 'operation.type',
    attributes: {
        'custom.attribute': 'value'
    }
}, async () => {
    // async 작업
    return await someAsyncOperation();
});
```

## 에러 레벨

적절한 심각도 수준 사용:

- **fatal**: 시스템 사용 불가 (데이터베이스 다운, 핵심 서비스 장애)
- **error**: 작업 실패, 즉시 주의 필요
- **warning**: 복구 가능한 문제, 성능 저하
- **info**: 정보성 메시지, 성공적인 작업
- **debug**: 상세 디버깅 정보 (개발 환경만)

## 필수 Context

```typescript
import * as Sentry from '@sentry/node';

Sentry.withScope((scope) => {
    // 사용 가능한 경우 항상 포함
    scope.setUser({ id: userId });
    scope.setTag('service', 'form'); // 또는 'email', 'users' 등
    scope.setTag('environment', process.env.NODE_ENV);

    // 작업별 context 추가
    scope.setContext('operation', {
        type: 'workflow.start',
        workflowCode: 'DHS_CLOSEOUT',
        entityId: 123
    });

    Sentry.captureException(error);
});
```

## 서비스별 통합

### Form Service

**위치**: `./blog-api/src/instrument.ts`

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

**주요 Helper**:
- `WorkflowSentryHelper` - Workflow 전용 에러
- `DatabasePerformanceMonitor` - DB 쿼리 추적
- `BaseController` - Controller error handling

### Email Service

**위치**: `./notifications/src/instrument.ts`

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

**주요 Helper**:
- `EmailSentryHelper` - 이메일 전용 에러
- `BaseController` - Controller error handling

## 설정 (config.ini)

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

## Sentry 통합 테스트

### Form Service 테스트 엔드포인트

```bash
# 기본 error capture 테스트
curl http://localhost:3002/blog-api/api/sentry/test-error

# Workflow error 테스트
curl http://localhost:3002/blog-api/api/sentry/test-workflow-error

# Database performance 테스트
curl http://localhost:3002/blog-api/api/sentry/test-database-performance

# Error boundary 테스트
curl http://localhost:3002/blog-api/api/sentry/test-error-boundary
```

### Email Service 테스트 엔드포인트

```bash
# 기본 error capture 테스트
curl http://localhost:3003/notifications/api/sentry/test-error

# Email 전용 error 테스트
curl http://localhost:3003/notifications/api/sentry/test-email-error

# Performance tracking 테스트
curl http://localhost:3003/notifications/api/sentry/test-performance
```

## Performance Monitoring

### 요구 사항

1. **모든 API 엔드포인트**는 트랜잭션 추적 필수
2. **100ms 초과 데이터베이스 쿼리**는 자동으로 플래그됨
3. **N+1 쿼리**는 감지 및 보고됨
4. **Cron jobs**는 실행 시간 추적 필수

### Transaction Tracking

```typescript
import * as Sentry from '@sentry/node';

// Express routes용 자동 트랜잭션 추적
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// 커스텀 작업용 수동 트랜잭션
const transaction = Sentry.startTransaction({
    op: 'operation.type',
    name: 'Operation Name',
});

try {
    // 작업 수행
} finally {
    transaction.finish();
}
```

## 피해야 할 일반적인 실수

❌ **절대** Sentry 없이 console.error만 사용하지 마세요
❌ **절대** 에러를 조용히 삼키지 마세요
❌ **절대** 에러 context에 민감한 데이터 노출하지 마세요
❌ **절대** context 없이 일반적인 에러 메시지 사용하지 마세요
❌ **절대** async 작업에서 error handling 건너뛰지 마세요
❌ **절대** cron jobs에서 instrument.ts를 첫 번째 줄에 import하는 것 잊지 마세요

## 구현 체크리스트

새 코드에 Sentry 추가 시:

- [ ] Sentry 또는 적절한 helper 임포트됨
- [ ] 모든 try/catch 블록이 Sentry에 캡처함
- [ ] 에러에 의미 있는 context 추가됨
- [ ] 적절한 에러 레벨 사용됨
- [ ] 에러 메시지에 민감한 데이터 없음
- [ ] 느린 작업에 performance tracking 추가됨
- [ ] Error handling 경로 테스트됨
- [ ] Cron jobs의 경우: instrument.ts가 첫 번째로 임포트됨

## 핵심 파일

### Form Service
- `/blog-api/src/instrument.ts` - Sentry 초기화
- `/blog-api/src/workflow/utils/sentryHelper.ts` - Workflow 에러
- `/blog-api/src/utils/databasePerformance.ts` - DB 모니터링
- `/blog-api/src/controllers/BaseController.ts` - Controller 베이스

### Email Service
- `/notifications/src/instrument.ts` - Sentry 초기화
- `/notifications/src/utils/EmailSentryHelper.ts` - 이메일 에러
- `/notifications/src/controllers/BaseController.ts` - Controller 베이스

### 설정
- `/blog-api/config.ini` - Form service 설정
- `/notifications/config.ini` - Email service 설정
- `/sentry.ini` - 공유 Sentry 설정

## 문서

- 전체 구현: `/dev/active/email-sentry-integration/`
- Form service 문서: `/blog-api/docs/sentry-integration.md`
- Email service 문서: `/notifications/docs/sentry-integration.md`

## 관련 Skills

- 데이터베이스 작업 전 **database-verification** 사용
- Workflow error context용 **workflow-builder** 사용
- 데이터베이스 error handling용 **database-scripts** 사용
