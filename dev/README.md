# Dev Docs 패턴

Claude Code 세션과 context 리셋 간에 프로젝트 context를 유지하기 위한 방법론입니다.

---

## 문제점

**Context 리셋은 모든 것을 잃어버립니다:**
- 구현 결정사항
- 주요 파일들과 그 목적
- 작업 진행 상황
- 기술적 제약사항
- 특정 접근법을 선택한 이유

**리셋 후, Claude는 모든 것을 다시 발견해야 합니다.**

---

## 해결책: 영구적인 Dev Docs

작업을 재개하는 데 필요한 모든 것을 담는 3-파일 구조:

```
dev/active/[task-name]/
├── [task-name]-plan.md      # 전략적 계획
├── [task-name]-context.md   # 주요 결정사항 & 파일들
└── [task-name]-tasks.md     # 체크리스트 형식
```

**이 파일들은 context 리셋에도 살아남습니다** - Claude가 이를 읽어 즉시 작업을 재개할 수 있습니다.

---

## 3-파일 구조

### 1. [task-name]-plan.md

**목적:** 구현을 위한 전략적 계획

**포함 내용:**
- 요약
- 현재 상태 분석
- 제안된 미래 상태
- 구현 단계
- 수락 기준이 포함된 상세 작업
- 위험 평가
- 성공 지표
- 일정 추정

**생성 시점:** 복잡한 작업 시작 시

**업데이트 시점:** 범위가 변경되거나 새로운 단계가 발견되었을 때

**예시:**
```markdown
# 기능 이름 - 구현 계획

## 요약
무엇을 구축하고 있으며 왜 하는가

## 현재 상태
현재 위치

## 구현 단계

### Phase 1: Infrastructure (2시간)
- Task 1.1: Database schema 설정
  - 수락: Schema 컴파일됨, 관계 올바름
- Task 1.2: Service 구조 생성
  - 수락: 모든 디렉토리 생성됨

### Phase 2: Core Functionality (3시간)
...
```

---

### 2. [task-name]-context.md

**목적:** 작업 재개를 위한 주요 정보

**포함 내용:**
- SESSION PROGRESS 섹션 (자주 업데이트!)
- 완료된 것 vs 진행 중인 것
- 주요 파일들과 그 목적
- 중요한 결정사항
- 발견된 기술적 제약사항
- 관련 파일 링크
- 빠른 재개 지침

**생성 시점:** 작업 시작 시

**업데이트 시점:** **자주** - 주요 결정, 완료, 또는 발견 후

**예시:**
```markdown
# 기능 이름 - Context

## SESSION PROGRESS (2025-10-29)

### ✅ 완료
- Database schema 생성됨 (User, Post, Comment 모델)
- PostController를 BaseController 패턴으로 구현
- Sentry 통합 작동 중

### 🟡 진행 중
- 비즈니스 로직이 있는 PostService 생성 중
- 파일: src/services/postService.ts

### ⚠️ 차단 요소
- 캐싱 전략 결정 필요

## 주요 파일

**src/controllers/PostController.ts**
- BaseController 확장
- Posts에 대한 HTTP 요청 처리
- PostService에 위임

**src/services/postService.ts** (진행 중)
- Post 작업을 위한 비즈니스 로직
- 다음: 캐싱 추가

## 빠른 재개
계속하려면:
1. 이 파일 읽기
2. PostService.createPost() 구현 계속하기
3. 남은 작업은 tasks 파일 참조
```

**중요:** 중요한 작업이 완료될 때마다 SESSION PROGRESS 섹션을 업데이트하세요!

---

### 3. [task-name]-tasks.md

**목적:** 진행 상황 추적을 위한 체크리스트

**포함 내용:**
- 논리적 섹션으로 나뉜 단계들
- 체크박스 형식의 작업들
- 상태 표시자 (✅/🟡/⏳)
- 수락 기준
- 빠른 재개 섹션

**생성 시점:** 작업 시작 시

**업데이트 시점:** 각 작업 완료 후 또는 새 작업 발견 시

**예시:**
```markdown
# 기능 이름 - Task 체크리스트

## Phase 1: 설정 ✅ 완료
- [x] Database schema 생성
- [x] Controllers 설정
- [x] Sentry 설정

## Phase 2: 구현 🟡 진행 중
- [x] PostController 생성
- [ ] PostService 생성 (진행 중)
- [ ] PostRepository 생성
- [ ] Zod로 검증 추가

## Phase 3: 테스팅 ⏳ 시작 안 함
- [ ] Service 단위 테스트
- [ ] 통합 테스트
- [ ] 수동 API 테스팅
```

---

## Dev Docs 사용 시점

**사용하는 경우:**
- ✅ 복잡한 여러 날 작업
- ✅ 많은 부분이 움직이는 기능
- ✅ 여러 세션에 걸칠 가능성이 있는 작업
- ✅ 신중한 계획이 필요한 작업
- ✅ 대규모 시스템 리팩토링

**건너뛰는 경우:**
- ❌ 간단한 버그 수정
- ❌ 단일 파일 변경
- ❌ 빠른 업데이트
- ❌ 사소한 수정

**경험 법칙:** 2시간 이상 걸리거나 여러 세션에 걸치면 dev docs를 사용하세요.

---

## Dev Docs를 사용한 워크플로우

### 새 작업 시작하기

1. **/dev-docs slash command 사용:**
   ```
   /dev-docs refactor authentication system
   ```

2. **Claude가 3개의 파일 생성:**
   - 요구사항 분석
   - 코드베이스 검사
   - 포괄적인 계획 생성
   - Context 및 tasks 파일 생성

3. **검토 및 조정:**
   - 계획이 합리적인지 확인
   - 누락된 고려사항 추가
   - 일정 추정 조정

### 구현 중

1. 전체 전략을 위해 **plan 참조**
2. **context.md를 자주 업데이트:**
   - 완료된 작업 표시
   - 결정사항 기록
   - 차단 요소 추가
3. 작업을 완료하면 tasks.md에서 **작업 체크**

### Context 리셋 후

1. **Claude가 3개 파일 모두 읽기**
2. 몇 초 만에 **완전한 상태 이해**
3. **정확히 중단했던 곳에서 재개**

무엇을 하고 있었는지 설명할 필요 없음 - 모두 문서화되어 있습니다!

---

## Slash Commands와 통합

### /dev-docs
**생성:** 작업을 위한 새 dev docs

**사용법:**
```
/dev-docs implement real-time notifications
```

**생성되는 것:**
- `dev/active/implement-real-time-notifications/`
  - implement-real-time-notifications-plan.md
  - implement-real-time-notifications-context.md
  - implement-real-time-notifications-tasks.md

### /dev-docs-update
**업데이트:** Context 리셋 전 기존 dev docs

**사용법:**
```
/dev-docs-update
```

**업데이트 내용:**
- 완료된 작업 표시
- 발견된 새 작업 추가
- 세션 진행 상황으로 context 업데이트
- 현재 상태 캡처

**사용 시점:** Context 제한에 가까워지거나 세션을 종료할 때

---

## 파일 구성

```
dev/
├── README.md              # 이 파일
├── active/                # 현재 작업
│   ├── task-1/
│   │   ├── task-1-plan.md
│   │   ├── task-1-context.md
│   │   └── task-1-tasks.md
│   └── task-2/
│       └── ...
└── archive/               # 완료된 작업 (선택)
    └── old-task/
        └── ...
```

**active/**: 진행 중인 작업
**archive/**: 완료된 작업 (참조용)

---

## 예시: 실제 사용

이 repository의 **dev/active/public-infrastructure-repo/**에서 실제 예시를 확인하세요:
- **plan.md** - 이 showcase 생성을 위한 700줄 이상의 전략적 계획
- **context.md** - 완료된 내용, 결정사항, 다음 할 일 추적
- **tasks.md** - 모든 단계와 작업의 체크리스트

이것은 이 showcase를 구축하는 데 사용된 실제 dev docs입니다!

---

## 모범 사례

### Context를 자주 업데이트

**나쁨:** 세션 끝에만 업데이트
**좋음:** 각 주요 마일스톤 후에 업데이트

**SESSION PROGRESS 섹션은 항상 현실을 반영해야 합니다:**
```markdown
## SESSION PROGRESS (YYYY-MM-DD)

### ✅ 완료 (완료된 모든 것 나열)
### 🟡 진행 중 (바로 지금 작업 중인 것)
### ⚠️ 차단 요소 (진행을 막고 있는 것)
```

### 작업을 실행 가능하게 만들기

**나쁨:** "인증 수정"
**좋음:** "AuthMiddleware.ts에서 JWT 토큰 검증 구현 (수락: 토큰 검증됨, 에러는 Sentry로)"

**포함할 내용:**
- 구체적인 파일 이름
- 명확한 수락 기준
- 다른 작업에 대한 의존성

### 계획을 최신 상태로 유지

범위가 변경되면:
- 계획 업데이트
- 새 단계 추가
- 일정 추정 조정
- 범위가 변경된 이유 기록

---

## Claude Code를 위해

**사용자가 dev docs 생성을 요청할 때:**

1. **가능하면 /dev-docs slash command 사용**
2. **또는 수동으로 생성:**
   - 작업 범위에 대해 질문
   - 관련 코드베이스 파일 분석
   - 포괄적인 계획 생성
   - Context 및 tasks 생성

3. **다음을 포함하여 계획 구성:**
   - 명확한 단계
   - 실행 가능한 작업
   - 수락 기준
   - 위험 평가

4. **Context 파일을 재개 가능하게 만들기:**
   - 맨 위에 SESSION PROGRESS
   - 빠른 재개 지침
   - 설명이 포함된 주요 파일 목록

**Dev docs에서 재개할 때:**

1. **3개 파일 모두 읽기** (plan, context, tasks)
2. **context.md부터 시작** - 현재 상태가 있음
3. **tasks.md 확인** - 완료된 것과 다음 할 일 확인
4. **plan.md 참조** - 전체 전략 이해

**자주 업데이트:**
- 작업 완료 즉시 표시
- 중요한 작업 후 SESSION PROGRESS 업데이트
- 발견된 새 작업 추가

---

## Dev Docs 수동 생성

/dev-docs command가 없는 경우:

**1. 디렉토리 생성:**
```bash
mkdir -p dev/active/your-task-name
```

**2. plan.md 생성:**
- 요약
- 구현 단계
- 상세 작업
- 일정 추정

**3. context.md 생성:**
- SESSION PROGRESS 섹션
- 주요 파일
- 중요한 결정사항
- 빠른 재개 지침

**4. tasks.md 생성:**
- 체크박스가 있는 단계
- [ ] 작업 형식
- 수락 기준

---

## 이점

**Dev docs 이전:**
- Context 리셋 = 처음부터 다시 시작
- 결정 이유를 잊어버림
- 진행 상황을 잃어버림
- 작업 반복

**Dev docs 이후:**
- Context 리셋 = 3개 파일 읽고 즉시 재개
- 결정사항 문서화됨
- 진행 상황 추적됨
- 반복 작업 없음

**절약된 시간:** Context 리셋당 몇 시간

---

## 다음 단계

1. 다음 복잡한 작업에서 **패턴 시도**
2. **/dev-docs** slash command 사용 (가능한 경우)
3. **자주 업데이트** - 특히 context.md
4. **실제 동작 확인** - dev/active/public-infrastructure-repo/ 둘러보기

**질문이 있으신가요?** [CLAUDE_INTEGRATION_GUIDE.md](../CLAUDE_INTEGRATION_GUIDE.md)를 참조하세요
