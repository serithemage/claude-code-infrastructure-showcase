# Skills

Context를 기반으로 자동 활성화되는 프로덕션 검증된 Claude Code skills입니다.

---

## Skills란 무엇인가요?

Skills는 Claude가 필요할 때 로드하는 모듈형 지식 베이스입니다. 다음을 제공합니다:
- 도메인별 가이드라인
- 모범 사례
- 코드 예시
- 피해야 할 안티패턴

**문제:** Skills는 기본적으로 자동 활성화되지 않습니다.

**해결책:** 이 showcase는 skills를 활성화시키는 hooks + 설정을 포함합니다.

---

## 사용 가능한 Skills

### skill-developer (Meta-Skill)
**목적:** Claude Code skills 생성 및 관리

**파일:** 리소스 파일 7개 (총 426줄)

**사용 시점:**
- 새 skills 생성
- Skill 구조 이해
- skill-rules.json 작업
- Skill 활성화 디버깅

**커스터마이징:** ✅ 불필요 - 그대로 복사

**[Skill 보기 →](skill-developer/)**

---

### backend-dev-guidelines
**목적:** Node.js/Express/TypeScript 개발 패턴

**파일:** 리소스 파일 12개 (메인 304줄 + 리소스)

**다루는 내용:**
- Layered architecture (Routes → Controllers → Services → Repositories)
- BaseController 패턴
- Prisma database 액세스
- Sentry error tracking
- Zod 검증
- UnifiedConfig 패턴
- Dependency injection
- 테스팅 전략

**사용 시점:**
- API routes 생성/수정
- Controllers 또는 services 구축
- Prisma를 사용한 database 작업
- Error tracking 설정

**커스터마이징:** ⚠️ skill-rules.json의 `pathPatterns`를 backend 디렉토리에 맞게 업데이트

**pathPatterns 예시:**
```json
{
  "pathPatterns": [
    "src/api/**/*.ts",       // src/api를 가진 단일 앱
    "backend/**/*.ts",       // Backend 디렉토리
    "services/*/src/**/*.ts" // Multi-service monorepo
  ]
}
```

**[Skill 보기 →](backend-dev-guidelines/)**

---

### frontend-dev-guidelines
**목적:** React/TypeScript/MUI v7 개발 패턴

**파일:** 리소스 파일 11개 (메인 398줄 + 리소스)

**다루는 내용:**
- 최신 React 패턴 (Suspense, lazy loading)
- 데이터 fetching을 위한 useSuspenseQuery
- MUI v7 styling (`size={{}}` prop을 가진 Grid)
- TanStack Router
- 파일 구성 (features/ 패턴)
- 성능 최적화
- TypeScript 모범 사례

**사용 시점:**
- React 컴포넌트 생성
- TanStack Query로 데이터 fetching
- MUI v7로 styling
- Routing 설정

**커스터마이징:** ⚠️ `pathPatterns` 업데이트 + React/MUI 사용 확인

**pathPatterns 예시:**
```json
{
  "pathPatterns": [
    "src/**/*.tsx",          // 단일 React 앱
    "frontend/src/**/*.tsx", // Frontend 디렉토리
    "apps/web/**/*.tsx"      // Monorepo web 앱
  ]
}
```

**참고:** 이 skill은 MUI v6→v7 비호환성을 방지하기 위해 **guardrail** (enforcement: "block")로 설정되어 있습니다.

**[Skill 보기 →](frontend-dev-guidelines/)**

---

### route-tester
**목적:** JWT cookie auth를 사용한 인증된 API routes 테스팅

**파일:** 메인 파일 1개 (389줄)

**다루는 내용:**
- JWT cookie 기반 인증 테스팅
- test-auth-route.js script 패턴
- Cookie 인증을 사용한 cURL
- Auth 이슈 디버깅
- POST/PUT/DELETE 작업 테스팅

**사용 시점:**
- API endpoints 테스팅
- 인증 디버깅
- Route 기능 검증

**커스터마이징:** ⚠️ JWT cookie auth 설정 필요

**먼저 물어보세요:** "JWT cookie 기반 인증을 사용하시나요?"
- YES인 경우: 복사하고 service URLs 커스터마이징
- NO인 경우: 건너뛰거나 auth 방법에 맞게 조정

**[Skill 보기 →](route-tester/)**

---

### error-tracking
**목적:** Sentry error tracking 및 모니터링 패턴

**파일:** 메인 파일 1개 (~250줄)

**다루는 내용:**
- Sentry v8 초기화
- Error capture 패턴
- Breadcrumbs 및 user context
- Performance 모니터링
- Express 및 React와의 통합

**사용 시점:**
- Error tracking 설정
- Exception 캡처
- Error context 추가
- 프로덕션 이슈 디버깅

**커스터마이징:** ⚠️ backend에 맞게 `pathPatterns` 업데이트

**[Skill 보기 →](error-tracking/)**

---

## 프로젝트에 Skill 추가하는 방법

### 빠른 통합

**Claude Code의 경우:**
```
사용자: "내 프로젝트에 backend-dev-guidelines skill을 추가해줘"

Claude는 다음을 해야 합니다:
1. 프로젝트 구조에 대해 질문
2. Skill 디렉토리 복사
3. 사용자의 경로로 skill-rules.json 업데이트
4. 통합 검증
```

전체 지침은 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)를 참조하세요.

### 수동 통합

**Step 1: Skill 디렉토리 복사**
```bash
cp -r claude-code-infrastructure-showcase/.claude/skills/backend-dev-guidelines \\
      your-project/.claude/skills/
```

**Step 2: skill-rules.json 업데이트**

없는 경우 생성:
```bash
cp claude-code-infrastructure-showcase/.claude/skills/skill-rules.json \\
   your-project/.claude/skills/
```

그 다음 프로젝트에 맞게 `pathPatterns` 커스터마이징:
```json
{
  "skills": {
    "backend-dev-guidelines": {
      "fileTriggers": {
        "pathPatterns": [
          "YOUR_BACKEND_PATH/**/*.ts"  // ← 이것을 업데이트하세요!
        ]
      }
    }
  }
}
```

**Step 3: 테스트**
- Backend 디렉토리에서 파일 편집
- Skill이 자동으로 활성화되어야 함

---

## skill-rules.json 설정

### 개요 및 목적

`skill-rules.json`은 여러 skills의 활성화 조건, 트리거, 권한 제어를 중앙에서 관리하는 핵심 설정 파일입니다. 프로젝트/조직 차원에서 일관된 개발 가이드라인과 안전장치를 구현합니다.

**Skills 활성화 기준:**
- 사용자 프롬프트의 **keywords** ("backend", "API", "route")
- **intentPatterns** (정규식으로 사용자 의도 매칭)
- **pathPatterns** (파일 경로 기반 - backend 파일 편집)
- **contentPatterns** (파일 내용 기반 - 코드에 Prisma 쿼리 포함)

---

### 전체 구조

```json
{
  "version": "1.0",
  "description": "Skill activation triggers for Claude Code",
  "skills": {
    "skill-name": {
      "type": "domain" | "guardrail",
      "enforcement": "suggest" | "block" | "warn",
      "priority": "critical" | "high" | "medium" | "low",
      "description": "Skill 설명",

      "promptTriggers": {
        "keywords": ["키워드", "목록"],
        "intentPatterns": ["(create|add).*?(route|API)"]
      },

      "fileTriggers": {
        "pathPatterns": ["backend/**/*.ts"],
        "pathExclusions": ["**/*.test.ts"],
        "contentPatterns": ["import.*Prisma"]
      },

      "blockMessage": "차단 시 표시할 메시지 (enforcement: block일 때)",

      "skipConditions": {
        "sessionSkillUsed": true,
        "fileMarkers": ["@skip-validation"],
        "envOverride": "SKIP_SKILL_NAME"
      }
    }
  }
}
```

---

### Enforcement 타입 상세

| Type | 동작 방식 | 사용 시점 | 예시 |
|------|-----------|-----------|------|
| **suggest** | 제안만 하고 차단하지 않음 | 일반 가이드라인, 모범 사례 | backend-dev-guidelines |
| **warn** | 경고 메시지 출력 후 진행 허용 | 주의가 필요한 작업 | 위험한 작업 경고 |
| **block** | 조건 충족 시 실행 차단 (guardrail) | 필수 준수 사항, 보안 | frontend-dev-guidelines (MUI v7 강제) |

**"block" 사용 권장:**
- Breaking changes 방지 (MUI v6→v7 비호환성)
- 중요한 database 작업
- 보안에 민감한 코드
- 필수 검증이 필요한 경우

**"suggest" 사용 권장:**
- 일반 모범 사례 안내
- 도메인별 가이드
- 코드 구성 제안
- 선택적 최적화

**"warn" 사용 권장:**
- 위험하지만 허용 가능한 작업
- 성능 영향이 있는 작업
- Deprecated 기능 사용 경고

---

### Priority Levels

여러 skills가 동시에 매칭될 때 우선순위를 결정합니다:

| Priority | 의미 | 사용 시점 |
|----------|------|-----------|
| **critical** | 항상 트리거 | 보안/필수 규칙 |
| **high** | 대부분의 매칭에서 트리거 | 중요한 가이드라인 |
| **medium** | 명확한 매칭에서 트리거 | 일반 패턴 |
| **low** | 명시적 매칭에서만 트리거 | 선택적 제안 |

**예시:**
- 보안 규칙 (block): `priority: "critical"`
- Frontend 가이드라인: `priority: "high"`
- 코드 스타일 제안: `priority: "low"`

---

### Triggers 상세 설명

#### promptTriggers
사용자의 프롬프트(입력)를 분석하여 skill 활성화:

```json
"promptTriggers": {
  "keywords": [
    "backend",
    "controller",
    "service",
    "API"
  ],
  "intentPatterns": [
    "(create|add|implement).*?(route|endpoint|API)",
    "(how to|best practice).*?(backend|controller)"
  ]
}
```

- **keywords**: 단순 키워드 매칭 (대소문자 무시)
- **intentPatterns**: 정규식으로 사용자 의도 파악 (더 정확한 매칭)

#### fileTriggers
파일 경로나 내용을 기반으로 skill 활성화:

```json
"fileTriggers": {
  "pathPatterns": [
    "backend/**/*.ts",
    "src/api/**/*.ts"
  ],
  "pathExclusions": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ],
  "contentPatterns": [
    "router\\.",
    "export.*Controller",
    "prisma\\."
  ]
}
```

**pathPatterns vs contentPatterns:**

| 타입 | 검사 대상 | 예시 | 용도 |
|------|----------|------|------|
| **pathPatterns** | 파일 경로/이름 | `"backend/**/*.ts"` | 특정 디렉토리/파일 타입 |
| **contentPatterns** | 파일 내용 | `"import.*Prisma"` | 특정 코드 패턴 감지 |
| **pathExclusions** | 제외할 경로 | `"**/*.test.ts"` | 테스트 파일 제외 |

---

### skipConditions (예외 처리)

특정 조건에서 규칙을 건너뛰는 메커니즘:

```json
"skipConditions": {
  "sessionSkillUsed": true,
  "fileMarkers": ["@skip-validation", "// @ignore-skill"],
  "envOverride": "SKIP_FRONTEND_GUIDELINES"
}
```

**각 조건 설명:**
- **sessionSkillUsed**: 세션에서 이미 skill을 사용한 경우 건너뛰기
- **fileMarkers**: 파일에 특정 주석이 있으면 건너뛰기 (개발자가 의도적으로 우회)
- **envOverride**: 환경변수 설정 시 건너뛰기

**사용 예시:**
```typescript
// @skip-validation
// 이 파일은 레거시 코드로 frontend-dev-guidelines를 따르지 않음
import { makeStyles } from '@material-ui/core';
```

---

### 모범 사례

#### 1. 명확한 Trigger 정의
```json
// ❌ 나쁨: 너무 광범위
"keywords": ["backend"]

// ✅ 좋음: 구체적
"keywords": ["backend", "controller", "service", "repository"],
"intentPatterns": ["(create|add|implement).*?(route|controller)"]
```

#### 2. pathPatterns 프로젝트 맞춤
```json
// ❌ 나쁨: 예제 경로 그대로
"pathPatterns": ["blog-api/src/**/*.ts"]

// ✅ 좋음: 실제 프로젝트 경로
"pathPatterns": [
  "src/api/**/*.ts",
  "backend/**/*.ts"
]
```

#### 3. 적절한 Enforcement 선택
```json
// 일반 가이드라인
"enforcement": "suggest"

// 필수 준수 사항 (보안, 호환성)
"enforcement": "block"

// 경고만 표시
"enforcement": "warn"
```

#### 4. skipConditions 활용
반복적인 경고를 방지하고 예외 상황을 명확히 처리:
```json
"skipConditions": {
  "sessionSkillUsed": true,  // 세션당 1회만 경고
  "fileMarkers": ["@legacy"]  // 레거시 코드 예외
}
```

---

### 커스터마이징 가이드

프로젝트에 맞게 조정해야 할 항목:

1. **pathPatterns**: 실제 프로젝트 디렉토리 구조에 맞게 변경
   ```json
   "pathPatterns": ["YOUR_PROJECT_PATH/**/*.ts"]
   ```

2. **keywords**: 도메인별 용어 추가
   ```json
   "keywords": ["your-domain-term", "project-specific"]
   ```

3. **intentPatterns**: 팀의 작업 패턴에 맞는 정규식
   ```json
   "intentPatterns": ["team-specific-pattern"]
   ```

---

### 실제 사용 예시

**Domain Skill (suggest):**
```json
"backend-dev-guidelines": {
  "type": "domain",
  "enforcement": "suggest",
  "priority": "high",
  "promptTriggers": {
    "keywords": ["backend", "API", "controller"],
    "intentPatterns": ["(create|add).*?(route|controller)"]
  },
  "fileTriggers": {
    "pathPatterns": ["backend/**/*.ts"],
    "contentPatterns": ["export.*Controller"]
  }
}
```

**Guardrail Skill (block):**
```json
"frontend-dev-guidelines": {
  "type": "guardrail",
  "enforcement": "block",
  "priority": "high",
  "blockMessage": "⚠️ BLOCKED - MUI v7 패턴 필요",
  "fileTriggers": {
    "pathPatterns": ["frontend/**/*.tsx"],
    "contentPatterns": ["makeStyles", "@material-ui/core"]
  },
  "skipConditions": {
    "sessionSkillUsed": true,
    "fileMarkers": ["@skip-validation"]
  }
}
```

---

## 자신만의 Skills 만들기

다음에 대한 완전한 가이드는 **skill-developer** skill을 참조하세요:
- Skill YAML frontmatter 구조
- Resource 파일 구성
- Trigger 패턴 설계
- Skill 활성화 테스팅

**빠른 템플릿:**
```markdown
---
name: my-skill
description: 이 skill이 하는 일
---

# My Skill 제목

## 목적
[이 skill이 존재하는 이유]

## Skill 사용 시점
[자동 활성화 시나리오]

## 빠른 참조
[주요 패턴 및 예시]

## Resource 파일
- [topic-1.md](resources/topic-1.md)
- [topic-2.md](resources/topic-2.md)
```

---

## 문제 해결

### Skill이 활성화되지 않음

**확인 사항:**
1. `.claude/skills/`에 skill 디렉토리가 있나요?
2. `skill-rules.json`에 skill이 나열되어 있나요?
3. `pathPatterns`가 파일과 일치하나요?
4. Hooks가 설치되어 작동하나요?
5. settings.json이 올바르게 설정되었나요?

**디버그:**
```bash
# Skill 존재 확인
ls -la .claude/skills/

# skill-rules.json 검증
cat .claude/skills/skill-rules.json | jq .

# Hooks가 실행 가능한지 확인
ls -la .claude/hooks/*.sh

# Hook을 수동으로 테스트
./.claude/hooks/skill-activation-prompt.sh
```

### Skill이 너무 자주 활성화됨

skill-rules.json 업데이트:
- 키워드를 더 구체적으로 만들기
- `pathPatterns` 범위 좁히기
- `intentPatterns`의 구체성 높이기

### Skill이 전혀 활성화되지 않음

skill-rules.json 업데이트:
- 더 많은 키워드 추가
- `pathPatterns` 범위 넓히기
- 더 많은 `intentPatterns` 추가

---

## Claude Code를 위해

**사용자를 위해 skill을 통합할 때:**

1. **먼저 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)를 읽으세요**
2. 프로젝트 구조에 대해 질문하세요
3. skill-rules.json에서 `pathPatterns` 커스터마이징
4. Skill 파일에 하드코딩된 경로가 없는지 확인
5. 통합 후 활성화 테스트

**일반적인 실수:**
- 예제 경로 유지 (blog-api/, frontend/)
- Monorepo vs 단일 앱에 대해 묻지 않음
- 커스터마이징 없이 skill-rules.json 복사

---

## 다음 단계

1. **간단하게 시작:** 작업과 일치하는 skill 하나 추가
2. **활성화 검증:** 관련 파일 편집, skill이 제안되어야 함
3. **더 추가:** 첫 번째 skill이 작동하면 다른 것들 추가
4. **커스터마이징:** 워크플로우에 따라 triggers 조정

**질문이 있으신가요?** 포괄적인 통합 지침은 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)를 참조하세요.
