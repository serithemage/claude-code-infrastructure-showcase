---
name: route-tester
description: cookie 기반 인증을 사용하여 프로젝트에서 인증된 routes 테스트. 이 skill은 API 엔드포인트 테스트, route 기능 검증, 인증 문제 디버깅 시 사용합니다. test-auth-route.js 사용 패턴과 mock 인증을 포함합니다.
---

# 프로젝트 Route Tester Skill

## 목적
이 skill은 cookie 기반 JWT 인증을 사용하여 프로젝트에서 인증된 routes를 테스트하기 위한 패턴을 제공합니다.

## 이 Skill 사용 시점
- 새 API 엔드포인트 테스트
- 변경 후 route 기능 검증
- 인증 문제 디버깅
- POST/PUT/DELETE 작업 테스트
- 요청/응답 데이터 검증

## 프로젝트 인증 개요

프로젝트에서 사용하는 것:
- **Keycloak** SSO용 (realm: yourRealm)
- **Cookie 기반 JWT** 토큰 (Bearer 헤더 아님)
- **Cookie 이름**: `refresh_token`
- **JWT 서명**: `config.ini`의 secret 사용

## 테스트 방법

### 방법 1: test-auth-route.js (권장)

`test-auth-route.js` 스크립트는 모든 인증 복잡성을 자동으로 처리합니다.

**위치**: `/root/git/your project_pre/scripts/test-auth-route.js`

#### 기본 GET 요청

```bash
node scripts/test-auth-route.js http://localhost:3000/blog-api/api/endpoint
```

#### JSON 데이터를 포함한 POST 요청

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

#### 스크립트가 하는 일

1. Keycloak에서 refresh token 가져오기
   - 사용자명: `testuser`
   - 비밀번호: `testpassword`
2. `config.ini`의 JWT secret으로 토큰 서명
3. cookie 헤더 생성: `refresh_token=<signed-token>`
4. 인증된 요청 수행
5. 수동으로 재현할 수 있는 정확한 curl 명령어 표시

#### 스크립트 출력

스크립트가 출력하는 것:
- 요청 세부 정보
- 응답 상태와 본문
- 수동 재현을 위한 curl 명령어

**참고**: 스크립트가 상세하므로 출력에서 실제 응답을 찾으세요.

### 방법 2: 토큰을 사용한 수동 curl

test-auth-route.js 출력의 curl 명령어 사용:

```bash
# 스크립트가 다음과 같이 출력합니다:
# 💡 curl로 수동 테스트하려면:
# curl -b "refresh_token=eyJhbGci..." http://localhost:3000/blog-api/api/endpoint

# 해당 curl 명령어를 복사하고 수정:
curl -X POST http://localhost:3000/blog-api/777/submit \
  -H "Content-Type: application/json" \
  -b "refresh_token=<스크립트_출력에서_토큰_복사>" \
  -d '{"your": "data"}'
```

### 방법 3: Mock 인증 (개발 환경만 - 가장 쉬움)

개발 환경에서 mock auth를 사용하여 Keycloak을 완전히 우회합니다.

#### 설정

```bash
# 서비스 .env 파일에 추가 (예: blog-api/.env)
MOCK_AUTH=true
MOCK_USER_ID=test-user
MOCK_USER_ROLES=admin,operations
```

#### 사용법

```bash
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-user" \
     -H "X-Mock-Roles: admin,operations" \
     http://localhost:3002/api/protected
```

#### Mock Auth 요구 사항

Mock auth는 다음 경우에만 작동합니다:
- `NODE_ENV`가 `development` 또는 `test`
- `mockAuth` middleware가 route에 추가됨
- 프로덕션에서는 절대 작동하지 않음 (보안 기능)

## 일반적인 테스트 패턴

### Form Submission 테스트

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

### Workflow 시작 테스트

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/start \
    POST \
    '{"workflowCode":"DHS_CLOSEOUT","entityType":"Submission","entityID":123}'
```

### Workflow Step 완료 테스트

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/step/complete \
    POST \
    '{"stepInstanceID":789,"answers":{"decision":"approved","comments":"Looks good"}}'
```

### Query Parameters가 있는 GET 테스트

```bash
node scripts/test-auth-route.js \
    "http://localhost:3002/api/workflows?status=active&limit=10"
```

### 파일 업로드 테스트

```bash
# 먼저 test-auth-route.js에서 토큰을 가져온 후:
curl -X POST http://localhost:5000/upload \
  -H "Content-Type: multipart/form-data" \
  -b "refresh_token=<TOKEN>" \
  -F "file=@/path/to/file.pdf" \
  -F "metadata={\"description\":\"Test file\"}"
```

## 하드코딩된 테스트 자격 증명

`test-auth-route.js` 스크립트가 사용하는 자격 증명:

- **사용자명**: `testuser`
- **비밀번호**: `testpassword`
- **Keycloak URL**: `config.ini`에서 (보통 `http://localhost:8081`)
- **Realm**: `yourRealm`
- **Client ID**: `config.ini`에서

## 서비스 포트

| 서비스 | 포트 | Base URL |
|---------|------|----------|
| Users   | 3000 | http://localhost:3000 |
| Projects| 3001 | http://localhost:3001 |
| Form    | 3002 | http://localhost:3002 |
| Email   | 3003 | http://localhost:3003 |
| Uploads | 5000 | http://localhost:5000 |

## Route 접두사

각 서비스의 `/src/app.ts`에서 route 접두사 확인:

```typescript
// blog-api/src/app.ts 예시
app.use('/blog-api/api', formRoutes);          // 접두사: /blog-api/api
app.use('/api/workflow', workflowRoutes);  // 접두사: /api/workflow
```

**전체 Route** = Base URL + 접두사 + Route 경로

예시:
- Base: `http://localhost:3002`
- 접두사: `/form`
- Route: `/777/submit`
- **전체 URL**: `http://localhost:3000/blog-api/777/submit`

## 테스트 체크리스트

Route 테스트 전:

- [ ] 서비스 식별 (form, email, users 등)
- [ ] 올바른 포트 찾기
- [ ] `app.ts`에서 route 접두사 확인
- [ ] 전체 URL 구성
- [ ] 요청 본문 준비 (POST/PUT인 경우)
- [ ] 인증 방법 결정
- [ ] 테스트 실행
- [ ] 응답 상태와 데이터 검증
- [ ] 해당되는 경우 데이터베이스 변경 확인

## 데이터베이스 변경 검증

데이터를 수정하는 routes 테스트 후:

```bash
# MySQL에 연결
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev

# 특정 테이블 확인
mysql> SELECT * FROM WorkflowInstance WHERE id = 123;
mysql> SELECT * FROM WorkflowStepInstance WHERE instanceId = 123;
mysql> SELECT * FROM WorkflowNotification WHERE recipientUserId = 'user-123';
```

## 실패한 테스트 디버깅

### 401 Unauthorized

**가능한 원인**:
1. 토큰 만료됨 (test-auth-route.js로 재생성)
2. 잘못된 cookie 형식
3. JWT secret 불일치
4. Keycloak이 실행 중이 아님

**해결책**:
```bash
# Keycloak이 실행 중인지 확인
docker ps | grep keycloak

# 토큰 재생성
node scripts/test-auth-route.js http://localhost:3002/api/health

# config.ini에 올바른 jwtSecret이 있는지 확인
```

### 403 Forbidden

**가능한 원인**:
1. 사용자에게 필요한 역할이 없음
2. 리소스 권한이 잘못됨
3. Route에 특정 권한 필요

**해결책**:
```bash
# admin 역할로 mock auth 사용
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-admin" \
     -H "X-Mock-Roles: admin" \
     http://localhost:3002/api/protected
```

### 404 Not Found

**가능한 원인**:
1. 잘못된 URL
2. 누락된 route 접두사
3. Route가 등록되지 않음

**해결책**:
1. `app.ts`에서 route 접두사 확인
2. Route 등록 확인
3. 서비스가 실행 중인지 확인 (`pm2 list`)

### 500 Internal Server Error

**가능한 원인**:
1. 데이터베이스 연결 문제
2. 필수 필드 누락
3. 검증 오류
4. 애플리케이션 오류

**해결책**:
1. 서비스 로그 확인 (`pm2 logs <service>`)
2. Sentry에서 오류 세부 정보 확인
3. 요청 본문이 예상 스키마와 일치하는지 확인
4. 데이터베이스 연결 확인

## auth-route-tester Agent 사용

변경 후 종합적인 route 테스트를 위해:

1. **영향받는 routes 식별**
2. **Route 정보 수집**:
   - 전체 route 경로 (접두사 포함)
   - 예상 POST 데이터
   - 검증할 테이블
3. **auth-route-tester agent 호출**

Agent가 수행하는 것:
- 적절한 인증으로 route 테스트
- 데이터베이스 변경 검증
- 응답 형식 확인
- 문제 보고

## 예시 테스트 시나리오

### 새 Route 생성 후

```bash
# 1. 유효한 데이터로 테스트
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"value1","field2":"value2"}'

# 2. 데이터베이스 검증
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev \
    -e "SELECT * FROM MyTable ORDER BY createdAt DESC LIMIT 1;"

# 3. 유효하지 않은 데이터로 테스트
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"invalid"}'

# 4. 인증 없이 테스트
curl http://localhost:3002/api/my-new-route
# 401을 반환해야 함
```

### Route 수정 후

```bash
# 1. 기존 기능이 여전히 작동하는지 테스트
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"existing":"data"}'

# 2. 새 기능 테스트
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"new":"field","existing":"data"}'

# 3. 하위 호환성 검증
# 이전 요청 형식으로 테스트 (해당되는 경우)
```

## 설정 파일

### config.ini (각 서비스)

```ini
[keycloak]
url = http://localhost:8081
realm = yourRealm
clientId = app-client

[jwt]
jwtSecret = your-jwt-secret-here
```

### .env (각 서비스)

```bash
NODE_ENV=development
MOCK_AUTH=true           # 선택: mock auth 활성화
MOCK_USER_ID=test-user   # 선택: 기본 mock 사용자
MOCK_USER_ROLES=admin    # 선택: 기본 mock 역할
```

## 핵심 파일

- `/root/git/your project_pre/scripts/test-auth-route.js` - 메인 테스트 스크립트
- `/blog-api/src/app.ts` - Form service routes
- `/notifications/src/app.ts` - Email service routes
- `/auth/src/app.ts` - Users service routes
- `/config.ini` - 서비스 설정
- `/.env` - 환경 변수

## 관련 Skills

- 데이터베이스 변경 검증에 **database-verification** 사용
- 캡처된 오류 확인에 **error-tracking** 사용
- Workflow route 테스트에 **workflow-builder** 사용
- 알림 전송 확인에 **notification-sender** 사용
