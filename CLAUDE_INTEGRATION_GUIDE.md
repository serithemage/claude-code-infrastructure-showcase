# Claude Integration Guide

**Claude Code를 위한 가이드:** 사용자가 이 showcase repository의 컴포넌트를 자신의 프로젝트에 통합해달라고 요청할 때, 다음 지침을 주의깊게 따르세요.

---

## 개요

이 repository는 Claude Code infrastructure 컴포넌트의 **레퍼런스 라이브러리**입니다. 사용자는 특정 부분을 자신의 프로젝트에 통합하는 것을 도와달라고 요청할 것입니다. 당신의 역할은:

1. **프로젝트 구조에 대한 명확한 질문하기**
2. **적절한 파일 복사하기**
3. **설정을 커스터마이징하기**
4. **통합이 올바르게 작동하는지 검증하기**

**핵심 원칙:** 프로젝트 구조를 가정하기 전에 항상 물어보세요. 한 프로젝트에서 작동하는 것이 다른 프로젝트에서는 작동하지 않을 수 있습니다.

---

## Tech Stack 호환성 확인

**중요:** Skill을 통합하기 전에 사용자의 tech stack이 skill 요구사항과 일치하는지 확인하세요.

### Frontend Skills

**frontend-dev-guidelines 요구사항:**
- React (18+)
- MUI v7
- TanStack Query
- TanStack Router
- TypeScript

**통합 전에 물어보세요:**
"React와 MUI v7를 사용하시나요?"

**아니오인 경우:**
```
frontend-dev-guidelines skill은 React + MUI v7 전용으로 설계되었습니다. 다음을 할 수 있습니다:
1. 이것을 템플릿으로 사용하여 [사용자의 stack]에 맞게 조정된 유사한 skill 생성
2. Framework에 독립적인 패턴 추출 (파일 구성, 성능 등)
3. 관련이 없다면 이 skill 건너뛰기

어떤 것을 선호하시나요?
```

### Backend Skills

**backend-dev-guidelines 요구사항:**
- Node.js/Express
- TypeScript
- Prisma ORM
- Sentry

**통합 전에 물어보세요:**
"Node.js와 Express, Prisma를 사용하시나요?"

**아니오인 경우:**
```
backend-dev-guidelines skill은 Express/Prisma용으로 설계되었습니다. 다음을 할 수 있습니다:
1. 이것을 템플릿으로 사용하여 [사용자의 stack]에 맞게 조정된 유사한 가이드라인 생성
2. 아키텍처 패턴 추출 (layered architecture는 모든 framework에 적용 가능)
3. 이 skill 건너뛰기

어떤 것을 선호하시나요?
```

### Skills That Are Tech-Agnostic

모든 tech stack에서 작동합니다:
- ✅ **skill-developer** - Meta-skill, 기술 요구사항 없음
- ✅ **route-tester** - JWT cookie 인증만 필요 (framework 독립적)
- ✅ **error-tracking** - Sentry는 대부분의 stack과 호환

---

## 일반 통합 패턴

사용자가 **"[컴포넌트]를 내 프로젝트에 추가해줘"**라고 말할 때:

1. 컴포넌트 타입 식별 (skill/hook/agent/command)
2. **TECH STACK 호환성 확인** (frontend/backend skills의 경우)
3. 프로젝트 구조에 대해 질문하기
4. 파일 복사 또는 사용자의 stack에 맞게 조정
5. 설정을 커스터마이징
6. 통합 검증
7. 다음 단계 제공

---

## Skills 통합하기

### 단계별 프로세스

**사용자가 skill을 요청할 때** (예: "backend-dev-guidelines 추가해줘"):

#### 1. 프로젝트 이해하기

**다음 질문들을 하세요:**
- "프로젝트 구조가 어떻게 되나요? 단일 앱, monorepo, 또는 multi-service?"
- "[backend/frontend] 코드는 어디에 있나요?"
- "어떤 frameworks/technologies를 사용하시나요?"

#### 2. Skill 복사하기

```bash
cp -r /path/to/showcase/.claude/skills/[skill-name] \\
      $CLAUDE_PROJECT_DIR/.claude/skills/
```

#### 3. skill-rules.json 처리하기

**존재 여부 확인:**
```bash
ls $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**없는 경우 (존재하지 않음):**
- showcase에서 템플릿 복사
- 사용자가 원하지 않는 skills 제거
- 프로젝트에 맞게 커스터마이징

**있는 경우 (존재함):**
- 현재 skill-rules.json 읽기
- 새 skill 항목 추가
- 기존 skills가 깨지지 않도록 신중히 병합

#### 4. Path 패턴 커스터마이징

**중요:** skill-rules.json의 `pathPatterns`를 사용자의 구조에 맞게 업데이트하세요:

**예시 - 사용자가 monorepo를 가진 경우:**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "packages/api/src/**/*.ts",
        "packages/server/src/**/*.ts",
        "apps/backend/**/*.ts"
      ]
    }
  }
}
```

**예시 - 사용자가 단일 backend를 가진 경우:**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "src/**/*.ts",
        "backend/**/*.ts"
      ]
    }
  }
}
```

**안전한 범용 패턴** (확실하지 않을 때):
```json
{
  "pathPatterns": [
    "**/*.ts",          // 모든 TypeScript 파일
    "src/**/*.ts",      // 일반적인 src 디렉토리
    "backend/**/*.ts"   // 일반적인 backend 디렉토리
  ]
}
```

#### 5. 통합 검증

```bash
# Skill이 복사되었는지 확인
ls -la $CLAUDE_PROJECT_DIR/.claude/skills/[skill-name]

# skill-rules.json 문법 검증
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
```

**사용자에게 알려주세요:** "[their-backend-path]에서 파일을 편집해보면 skill이 활성화될 것입니다."

---

### Skill별 참고사항

#### backend-dev-guidelines
- **기술 요구사항:** Node.js/Express, Prisma, TypeScript, Sentry
- **질문하세요:** "Express와 Prisma를 사용하시나요?" "backend 코드는 어디에 있나요?"
- **다른 stack인 경우:** 이것을 템플릿으로 사용하여 조정하겠다고 제안
- **커스터마이징:** pathPatterns
- **예시 경로:** `api/`, `server/`, `backend/`, `services/*/src/`
- **조정 팁:** 아키텍처 패턴 (Routes→Controllers→Services)은 대부분의 frameworks에 적용됨

#### frontend-dev-guidelines
- **기술 요구사항:** React 18+, MUI v7, TanStack Query/Router, TypeScript
- **질문하세요:** "React와 MUI v7를 사용하시나요?" "frontend 코드는 어디에 있나요?"
- **다른 stack인 경우:** 조정된 버전 생성 제안 (Vue, Angular 등)
- **커스터마이징:** pathPatterns + 모든 framework별 예제
- **예시 경로:** `frontend/`, `client/`, `web/`, `apps/web/src/`
- **조정 팁:** 파일 구성과 성능 패턴은 적용되지만, 컴포넌트 코드는 적용 안됨

#### route-tester
- **기술 요구사항:** JWT cookie 기반 인증 (framework 독립적)
- **질문하세요:** "JWT cookie 기반 인증을 사용하시나요?"
- **아니오인 경우:** "이 skill은 JWT cookies용으로 설계되었습니다. [사용자의 auth 타입]에 맞게 조정하거나 건너뛸까요?"
- **커스터마이징:** Service URLs, 인증 패턴
- **호환:** JWT cookies를 사용하는 모든 backend framework

#### error-tracking
- **기술 요구사항:** Sentry (대부분의 backends와 호환)
- **질문하세요:** "Sentry를 사용하시나요?" "backend 코드는 어디에 있나요?"
- **Sentry를 사용하지 않는 경우:** "[사용자의 error tracking]을 위한 템플릿으로 사용하시겠어요?"
- **커스터마이징:** pathPatterns
- **조정 팁:** Error tracking 철학은 다른 도구로 적용 가능 (Rollbar, Bugsnag 등)

#### skill-developer
- **기술 요구사항:** 없음!
- **그대로 복사** - meta-skill, 완전히 범용, 모든 tech stack의 skill 생성을 가르침

---

## 다른 Tech Stack에 맞게 Skills 조정하기

사용자의 tech stack이 skill 요구사항과 다를 때 다음 옵션들이 있습니다:

### 옵션 1: 기존 Skill 조정 (권장)

**언제 사용:** 사용자가 다른 기술을 위한 유사한 가이드라인을 원할 때

**프로세스:**
1. **시작점으로 skill 복사:**
   ```bash
   cp -r showcase/.claude/skills/frontend-dev-guidelines \\
         $CLAUDE_PROJECT_DIR/.claude/skills/vue-dev-guidelines
   ```

2. **변경이 필요한 것 식별:**
   - Framework별 코드 예제 (React → Vue)
   - Library APIs (MUI → Vuetify/PrimeVue)
   - Import 문
   - 컴포넌트 패턴

3. **적용되는 것 유지:**
   - 파일 구성 원칙
   - 성능 최적화 전략
   - TypeScript 표준
   - 일반 모범 사례

4. **체계적으로 예제 교체:**
   - 사용자의 stack에서 동등한 패턴 물어보기
   - 코드 예제를 사용자의 framework에 맞게 업데이트
   - 전체 구조와 섹션 유지

5. **Skill 이름과 triggers 업데이트:**
   - Skill 이름을 적절히 변경
   - skill-rules.json triggers를 사용자의 stack에 맞게 업데이트
   - 활성화 테스트

**예시 - Vue에 맞게 frontend-dev-guidelines 조정:**
```
React skill 구조를 기반으로 vue-dev-guidelines를 생성하겠습니다:
- React.FC → Vue defineComponent로 교체
- useSuspenseQuery → Vue composables로 교체
- MUI components → [사용자의 component library]로 교체
- 유지: 파일 구성, 성능 패턴, TypeScript 가이드라인

몇 분 정도 걸릴 것 같습니다. 괜찮으신가요?
```

### 옵션 2: Framework에 독립적인 패턴 추출

**언제 사용:** Stack이 매우 다르지만 핵심 원칙은 적용 가능할 때

**프로세스:**
1. 기존 skill을 읽기
2. Framework에 독립적인 패턴 식별:
   - Layered architecture (backend)
   - 파일 구성 전략
   - 성능 최적화 원칙
   - 테스팅 전략
   - Error 처리 철학

3. 해당 패턴만으로 새 skill 생성
4. 사용자가 나중에 framework별 예제 추가 가능

**예시:**
```
backend-dev-guidelines는 Express를 사용하지만, layered architecture
(Routes → Controllers → Services → Repositories)는 Django에서도 작동합니다.

다음 내용으로 skill을 생성할 수 있습니다:
- Layered architecture 패턴
- 관심사 분리 원칙
- Error 처리 모범 사례
- 테스팅 전략

그런 다음 패턴을 확립하면서 Django별 예제를 추가할 수 있습니다.
```

### 옵션 3: 레퍼런스로만 사용

**언제 사용:** 조정하기에는 너무 다르지만 사용자가 영감을 원할 때

**프로세스:**
1. 사용자가 기존 skill을 둘러봄
2. 처음부터 새 skill 생성을 도와줌
3. 기존 skill의 구조를 템플릿으로 사용
4. 모듈형 패턴 따르기 (main + resource 파일)

### Tech Stack 간에 일반적으로 적용되는 것

**아키텍처 & 구성:**
- ✅ Layered architecture (Routes/Controllers/Services 패턴)
- ✅ 관심사 분리
- ✅ 파일 구성 전략 (features/ 패턴)
- ✅ Progressive disclosure (main + resource 파일)
- ✅ 데이터 액세스를 위한 Repository 패턴

**개발 관행:**
- ✅ Error 처리 철학
- ✅ Input 검증의 중요성
- ✅ 테스팅 전략
- ✅ 성능 최적화 원칙
- ✅ TypeScript 모범 사례

**Framework별 코드:**
- ❌ React hooks → Vue/Angular로 적용 안됨
- ❌ MUI components → 다른 component libraries
- ❌ Prisma queries → 다른 ORM 문법
- ❌ Express middleware → 다른 framework 패턴
- ❌ Routing 구현 → Framework별

### 조정 vs 건너뛰기 권장 시점

**조정을 권장하는 경우:**
- 사용자가 자신의 stack을 위한 유사한 가이드라인을 원할 때
- 핵심 패턴이 적용될 때 (layered architecture 등)
- 사용자가 framework별 예제를 도와줄 시간이 있을 때

**건너뛰기를 권장하는 경우:**
- Stack이 완전히 다를 때
- 사용자가 해당 패턴이 필요 없을 때
- 조정하는 데 시간이 너무 오래 걸릴 때
- 사용자가 처음부터 생성하는 것을 선호할 때

---

## Hooks 통합하기

### 필수 Hooks (항상 안전하게 복사 가능)

#### skill-activation-prompt (UserPromptSubmit)

**목적:** 사용자 프롬프트를 기반으로 skills를 자동 제안

**통합 (커스터마이징 불필요):**

```bash
# 두 파일 모두 복사
cp showcase/.claude/hooks/skill-activation-prompt.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/
cp showcase/.claude/hooks/skill-activation-prompt.ts \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 실행 가능하게 만들기
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# 필요시 dependencies 설치
if [ -f "showcase/.claude/hooks/package.json" ]; then
  cp showcase/.claude/hooks/package.json \\
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && npm install
fi
```

**settings.json에 추가:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**이 hook은 완전히 범용적입니다** - 어디서나 작동, 커스터마이징 불필요!

#### post-tool-use-tracker (PostToolUse)

**목적:** Context 관리를 위한 파일 변경 추적

**통합 (커스터마이징 불필요):**

```bash
# 파일 복사
cp showcase/.claude/hooks/post-tool-use-tracker.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 실행 가능하게 만들기
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
```

**settings.json에 추가:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**이 hook은 완전히 범용적입니다** - 프로젝트 구조를 자동 감지!

---

### 선택적 Hooks (많은 커스터마이징 필요)

#### tsc-check.sh 및 trigger-build-resolver.sh (Stop hooks)

⚠️ **경고:** 이 hooks는 특정 multi-service monorepo 구조용으로 설정되었습니다.

**통합 전에 물어보세요:**
1. "여러 TypeScript services가 있는 monorepo가 있나요?"
2. "service 디렉토리 이름은 무엇인가요?"
3. "tsconfig.json 파일들은 어디에 있나요?"

**단순 프로젝트의 경우 (단일 service):**
- **이 hooks 건너뛰기 권장**
- 단일 service 프로젝트에는 과한 설정
- 사용자가 대신 `tsc --noEmit`을 수동으로 실행 가능

**복잡한 프로젝트의 경우 (multi-service monorepo):**

1. 파일 복사
2. **반드시 편집** tsc-check.sh - 이 섹션 찾기:
```bash
case "$repo" in
    email|exports|form|frontend|projects|uploads|users|utilities|events|database)
        echo "$repo"
        return 0
        ;;
esac
```

3. 사용자의 실제 service 이름으로 교체:
```bash
case "$repo" in
    api|web|auth|payments|notifications)  # ← 사용자의 services
        echo "$repo"
        return 0
        ;;
esac
```

4. settings.json에 추가하기 전에 수동으로 테스트:
```bash
./.claude/hooks/tsc-check.sh
```

**중요:** 이 hook이 실패하면 Stop 이벤트를 차단합니다. 설정에서 작동한다고 확신할 때만 추가하세요.

---

## Agents 통합하기

**Agents는 독립형입니다** - 통합이 가장 쉽습니다!

### 표준 Agent 통합

```bash
# Agent 파일 복사
cp showcase/.claude/agents/[agent-name].md \\
   $CLAUDE_PROJECT_DIR/.claude/agents/
```

**끝입니다!** Agents는 즉시 작동하며 설정이 필요 없습니다.

### 하드코딩된 경로 확인

일부 agents는 경로를 참조할 수 있습니다. **복사하기 전에 agent 파일을 읽고 다음을 확인하세요:**

- `~/git/old-project/` → `$CLAUDE_PROJECT_DIR` 또는 `.`로 변경
- `/root/git/project/` → `$CLAUDE_PROJECT_DIR` 또는 `.`로 변경
- 하드코딩된 스크린샷 경로 → 사용자에게 스크린샷을 어디에 저장할지 물어보기

**발견된 경우 업데이트:**
```bash
sed -i 's|~/git/old-project/|.|g' $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
sed -i 's|/root/git/.*PROJECT.*DIR|$CLAUDE_PROJECT_DIR|g' \\
    $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
```

### Agent별 참고사항

**auth-route-tester / auth-route-debugger:**
- 사용자 프로젝트에 JWT cookie 기반 인증 필요
- 질문: "Auth에 JWT cookies를 사용하시나요?"
- 아니오인 경우: "이 agents는 JWT cookie auth용입니다. 건너뛰거나 조정할까요?"

**frontend-error-fixer:**
- 스크린샷 경로를 참조할 수 있음
- 질문: "스크린샷을 어디에 저장할까요?"

**다른 모든 agents:**
- 그대로 복사, 완전히 범용적임

---

## Slash Commands 통합하기

```bash
# Command 파일 복사
cp showcase/.claude/commands/[command].md \\
   $CLAUDE_PROJECT_DIR/.claude/commands/
```

### 경로 커스터마이징

Commands가 dev docs 경로를 참조할 수 있습니다. **확인 및 업데이트:**

**dev-docs 및 dev-docs-update:**
- `dev/active/` 경로 참조 찾기
- 질문: "dev documentation을 어디에 저장하고 싶으신가요?"
- Command 파일의 경로 업데이트

**route-research-for-testing:**
- Service 경로를 참조할 수 있음
- API 구조에 대해 질문

---

## 일반적인 패턴 & 모범 사례

### 패턴: 프로젝트 구조에 대해 질문하기

**가정하지 마세요:**
- ❌ "blog-api service에 이것을 추가하겠습니다"
- ❌ "frontend 디렉토리에 맞게 설정하겠습니다"

**질문하세요:**
- ✅ "프로젝트 구조가 어떻게 되나요? Monorepo 또는 단일 앱?"
- ✅ "backend 코드는 어디에 있나요?"
- ✅ "workspaces를 사용하거나 여러 services가 있나요?"

### 패턴: skill-rules.json 커스터마이징

**사용자가 workspaces가 있는 monorepo를 가진 경우:**
```json
{
  "pathPatterns": [
    "packages/*/src/**/*.ts",
    "apps/*/src/**/*.tsx"
  ]
}
```

**사용자가 Nx monorepo를 가진 경우:**
```json
{
  "pathPatterns": [
    "apps/api/src/**/*.ts",
    "libs/*/src/**/*.ts"
  ]
}
```

**사용자가 간단한 구조를 가진 경우:**
```json
{
  "pathPatterns": [
    "src/**/*.ts",
    "backend/**/*.ts"
  ]
}
```

### 패턴: settings.json 통합

**showcase의 settings.json을 직접 복사하지 마세요!**

대신 필요한 섹션을 **추출하고 병합**하세요:

1. 기존 settings.json 읽기
2. 원하는 hook 설정 추가
3. 기존 설정 보존

**병합 예시:**
```json
{
  // ... 기존 설정 ...
  "hooks": {
    // ... 기존 hooks ...
    "UserPromptSubmit": [  // ← 이 섹션 추가
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 검증 체크리스트

통합 후 **다음 항목들을 검증하세요:**

```bash
# 1. Hooks가 실행 가능한지
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
# 다음과 같이 표시되어야 함: -rwxr-xr-x

# 2. skill-rules.json이 유효한 JSON인지
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
# 에러 없이 파싱되어야 함

# 3. Hook dependencies 설치됨 (TypeScript hooks인 경우)
ls $CLAUDE_PROJECT_DIR/.claude/hooks/node_modules/
# package.json이 있으면 패키지들이 표시되어야 함

# 4. Settings.json이 유효한 JSON인지
cat $CLAUDE_PROJECT_DIR/.claude/settings.json | jq .
# 에러 없이 파싱되어야 함
```

**그 다음 사용자에게 테스트 요청:**
- "[relevant-path]에서 파일을 편집해보세요 - skill이 활성화되어야 합니다"
- "[topic]에 대해 질문해보세요 - skill을 제안해야 합니다"

---

## 피해야 할 일반적인 실수

### ❌ 하지 마세요: settings.json을 그대로 복사
**이유:** Stop hooks가 존재하지 않는 services를 참조함
**대신:** UserPromptSubmit과 PostToolUse 섹션만 추출

### ❌ 하지 마세요: 예제 service 이름 유지
**이유:** 사용자는 blog-api, auth-service 등이 없음
**대신:** 실제 구조에 대해 물어보고 업데이트

### ❌ 하지 마세요: Hooks를 실행 가능하게 만드는 것 건너뛰기
**이유:** 실행 권한 없이는 hooks가 실행되지 않음
**대신:** 복사 후 항상 `chmod +x` 실행

### ❌ 하지 마세요: Monorepo 구조 가정
**이유:** 대부분의 프로젝트는 단일 service
**대신:** 먼저 물어본 다음 커스터마이징

### ❌ 하지 마세요: 모든 skills를 한 번에 추가
**이유:** 압도적이며 모두 관련이 없을 수 있음
**대신:** 작업과 관련된 skills가 무엇인지 물어보기

### ❌ 하지 마세요: 테스트 없이 Stop hooks 복사
**이유:** 실패할 경우 Stop 이벤트를 차단할 수 있음
**대신:** 먼저 수동으로 테스트, 작동할 때만 추가

---

## 예시 통합 대화

### 예시 1: Backend Skill 추가

**사용자:** "backend-dev-guidelines skill을 내 프로젝트에 추가해줘"

**Claude는 다음과 같이 응답해야 합니다:**
```
backend-dev-guidelines skill을 추가하겠습니다. 먼저 프로젝트 구조를
이해해야 합니다:

1. 여러 services가 있는 monorepo인가요, 아니면 단일 backend 애플리케이션인가요?
2. backend 코드는 어디에 있나요? (예: src/, backend/, server/)
3. Express.js를 사용하시나요?
```

**사용자:** "단일 앱이고, 코드는 src/api/에 있어. Express 사용해"

**Claude는 다음을 해야 합니다:**
1. Skill 디렉토리 복사
2. skill-rules.json을 pathPattern: `"src/api/**/*.ts"`로 업데이트
3. 올바르게 복사되었는지 검증
4. 응답:
```
✅ backend-dev-guidelines skill 추가 완료!

설정:
- src/api/에서 파일 편집 시 skill 트리거됨
- "controller", "service", "route", "API" 같은 키워드로 활성화됨

테스트: src/api/에서 파일을 편집하면 skill 제안이 표시될 것입니다.
```

### 예시 2: Hooks 설정

**사용자:** "skill 활성화 hooks를 설정해줘"

**Claude는 다음과 같이 응답해야 합니다:**
```
skill 자동 활성화 시스템을 설정하겠습니다. 두 개의 hooks가 필요합니다:
1. skill-activation-prompt - 프롬프트를 기반으로 skills 제안
2. post-tool-use-tracker - 파일 변경 추적

.claude/settings.json 파일이 이미 있나요, 아니면 새로 만들어야 하나요?
```

**사용자:** "이미 settings.json이 있어"

**Claude는 다음을 해야 합니다:**
1. settings.json 읽기
2. 두 hook 파일 복사
3. hooks 디렉토리에서 npm install 실행
4. settings.json에 hook 설정 추가 (기존 설정 보존)
5. Hooks를 실행 가능하게 만들기
6. 응답:
```
✅ Skill 활성화 hooks 설치 완료!

수행한 작업:
- skill-activation-prompt hook 복사
- post-tool-use-tracker hook 복사
- Dependencies 설치
- settings.json에 추가

테스트: "backend 개발 도와줘"라고 물어보면 backend-dev-guidelines skill을
제안할 것입니다.
```

### 예시 3: Agent 추가

**사용자:** "code-architecture-reviewer agent를 추가해줘"

**Claude는 다음과 같이 응답해야 합니다:**
```
code-architecture-reviewer agent는 아키텍처 문제와 모범 사례에 대해 코드를
리뷰합니다. 이 agent는 독립형이며 설정이 필요 없습니다.

.claude/agents/ 디렉토리에 복사하겠습니다.
```

**그 다음:**
1. Agent 파일 복사
2. 응답:
```
✅ code-architecture-reviewer agent 추가 완료!

다음으로 Task tool을 실행하여 사용하세요:
- subagent_type: "code-architecture-reviewer"
- prompt: 리뷰할 코드 설명

Agent가 코드를 리뷰하고 아키텍처 피드백을 제공할 것입니다.
```

### 예시 4: Tech Stack 불일치

**사용자:** "Vue 프로젝트에 frontend-dev-guidelines skill을 추가해줘"

**Claude는 다음과 같이 응답해야 합니다:**
```
Vue를 사용하시는군요. frontend-dev-guidelines skill은 React + MUI v7 전용으로
설계되었습니다. 몇 가지 옵션이 있습니다:

옵션 1 (권장): React skill을 템플릿으로 사용하여 vue-dev-guidelines skill을
생성할 수 있습니다. 다음을 조정하겠습니다:
- Vue용 컴포넌트 패턴 (Composition API, defineComponent)
- MUI 예제를 사용자의 component library로 교체
- Vue 패턴에 맞게 data fetching 조정
- 유지: 파일 구성, 성능 원칙, TypeScript 표준

옵션 2: React별 코드 없이 framework에 독립적인 패턴만 추출 (파일 구성,
성능, 테스팅)

옵션 3: 건너뛰고 처음부터 Vue 가이드라인 생성

어떤 것을 선호하시나요?
```

**사용자:** "옵션 1이 좋을 것 같아, Vuetify 사용해"

**Claude는 다음을 해야 합니다:**
1. frontend-dev-guidelines를 vue-dev-guidelines로 복사
2. 각 resource 파일 처리
3. React 패턴을 Vue Composition API로 교체
4. MUI 컴포넌트를 Vuetify로 교체
5. skill-rules.json을 Vue별 triggers로 업데이트 (.vue 파일)
6. 응답:
```
✅ Vue 3 + Vuetify에 맞게 조정된 vue-dev-guidelines skill 생성 완료!

변경된 내용:
- React.FC 컴포넌트 → Vue defineComponent
- MUI v7 예제 → Vuetify 컴포넌트
- TanStack Query → 사용자의 data fetching 접근법
- 파일 패턴 → .vue 파일

유지된 내용:
- 파일 구성 (features/ 패턴)
- 성능 최적화 전략
- TypeScript 표준
- Loading/error state 원칙

.vue 파일을 편집해보세요 - skill이 활성화될 것입니다.
```

---

## 빠른 참조 테이블

### 커스터마이징이 필요한 것은?

| 컴포넌트 | 기술 요구사항 | 커스터마이징 | 질문할 내용 |
|-----------|------------------|--------------|-------------|
| **skill-developer** | 없음 | ✅ 불필요 | 그대로 복사 |
| **backend-dev-guidelines** | Express/Prisma/Node | ⚠️ 경로 + 기술 확인 | "Express/Prisma 사용?" "backend 위치?" |
| **frontend-dev-guidelines** | React/MUI v7 | ⚠️⚠️ 경로 + framework | "React/MUI v7 사용?" "frontend 위치?" |
| **route-tester** | JWT cookies | ⚠️ Auth + 경로 | "JWT cookie auth?" |
| **error-tracking** | Sentry | ⚠️ 경로 | "Sentry 사용?" "backend 위치?" |
| **skill-activation-prompt** | ✅ 없음 | 그대로 복사 |
| **post-tool-use-tracker** | ✅ 없음 | 그대로 복사 |
| **tsc-check** | ⚠️⚠️⚠️ 무거움 | "Monorepo 또는 단일 service?" |
| **모든 agents** | ✅ 최소 | 경로 확인 |
| **모든 commands** | ⚠️ 경로 | "dev docs 위치?" |

### 건너뛰기를 권장하는 시점

| 컴포넌트 | 다음인 경우 건너뛰기... |
|-----------|-----------|
| **tsc-check hooks** | 단일 service 프로젝트 또는 다른 build 설정 |
| **route-tester** | JWT cookie 인증 미사용 |
| **frontend-dev-guidelines** | React + MUI 미사용 |
| **auth agents** | JWT cookie auth 미사용 |

---

## Claude를 위한 최종 팁

**사용자가 "모두 추가해줘"라고 말할 때:**
- 필수 요소부터 시작: skill-activation hooks + 관련 skills 1-2개
- 모든 5개 skills + 10개 agents로 압도하지 마세요
- 실제로 필요한 것이 무엇인지 물어보세요

**무언가 작동하지 않을 때:**
- 검증 체크리스트 확인
- 경로가 구조와 일치하는지 검증
- Hooks를 수동으로 테스트
- JSON 문법 오류 확인

**사용자가 확신이 없을 때:**
- skill-activation hooks만으로 시작하도록 권장
- Backend 또는 frontend skill 추가 (사용하는 것)
- 나중에 필요에 따라 더 추가

**항상 수행 중인 작업 설명:**
- 실행 중인 명령어 표시
- 질문하는 이유 설명
- 통합 후 명확한 다음 단계 제공

---

**기억하세요:** 이것은 레퍼런스 라이브러리이지 동작하는 애플리케이션이 아닙니다. 당신의 역할은 사용자가 자신의 특정 프로젝트 구조에 맞게 컴포넌트를 선별하고 조정하도록 돕는 것입니다.
