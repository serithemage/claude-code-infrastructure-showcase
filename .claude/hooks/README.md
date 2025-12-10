# Hooks

Skills 자동 활성화, 파일 추적, 유효성 검사를 가능하게 하는 Claude Code hooks입니다.

---

## Hooks란?

Hooks는 Claude 워크플로우의 특정 시점에 실행되는 스크립트입니다:
- **UserPromptSubmit**: 사용자가 프롬프트를 제출할 때
- **PreToolUse**: 도구가 실행되기 전
- **PostToolUse**: 도구가 완료된 후
- **Stop**: 사용자가 중지를 요청할 때

**핵심 인사이트:** Hooks는 프롬프트를 수정하고, 작업을 차단하고, 상태를 추적할 수 있어 Claude가 단독으로 할 수 없는 기능을 가능하게 합니다.

---

## 파일 구조

```
.claude/hooks/
├── skill-activation-prompt.sh    # Shell wrapper
├── skill-activation-prompt.ts    # TypeScript 메인 로직
├── post-tool-use-tracker.sh      # 파일 변경 추적
├── tsc-check.sh                  # TypeScript 컴파일 검사 (PostToolUse)
├── trigger-build-resolver.sh     # 빌드 에러 해결 agent 트리거 (Stop)
├── error-handling-reminder.sh    # Shell wrapper
├── error-handling-reminder.ts    # 에러 핸들링 셀프체크
├── stop-build-check-enhanced.sh  # 향상된 Stop 빌드 검사
├── package.json                  # TypeScript 의존성
├── tsconfig.json                 # TypeScript 설정
├── CONFIG.md                     # 추가 설정 가이드
└── README.md                     # 이 문서
```

---

## 필수 Hooks (여기서 시작)

### skill-activation-prompt (UserPromptSubmit)

**목적:** 사용자 프롬프트와 파일 컨텍스트를 기반으로 관련 skills를 자동으로 제안

**파일 구성:**
- `skill-activation-prompt.sh` - npx tsx로 TypeScript 실행하는 wrapper
- `skill-activation-prompt.ts` - 실제 로직 구현

**작동 방식:**

```
[사용자 프롬프트] → [Hook 실행] → [skill-rules.json 로드] → [패턴 매칭] → [제안 출력]
```

1. stdin으로 hook input JSON 수신 (session_id, prompt 등)
2. `$CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json` 로드
3. 사용자 프롬프트를 각 skill의 트리거 패턴과 매칭:
   - **keywords**: 단순 키워드 포함 여부 검사 (대소문자 무시)
   - **intentPatterns**: 정규식 기반 의도 패턴 매칭
4. 매칭된 skills를 priority별로 그룹화하여 출력:
   - `critical` → ⚠️ CRITICAL SKILLS (REQUIRED)
   - `high` → 📚 RECOMMENDED SKILLS
   - `medium` → 💡 SUGGESTED SKILLS
   - `low` → 📌 OPTIONAL SKILLS

**출력 예시:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 SKILL ACTIVATION CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 RECOMMENDED SKILLS:
  → backend-dev-guidelines

💡 SUGGESTED SKILLS:
  → error-tracking

ACTION: Use Skill tool BEFORE responding
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**필수인 이유:** Skills가 자동 활성화되게 만드는 핵심 hook입니다.

**통합 방법:**
```bash
# 두 파일 모두 복사
cp skill-activation-prompt.sh your-project/.claude/hooks/
cp skill-activation-prompt.ts your-project/.claude/hooks/

# 실행 권한 부여
chmod +x your-project/.claude/hooks/skill-activation-prompt.sh

# 의존성 설치
cd your-project/.claude/hooks
npm install
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

**커스터마이징:** ✅ 불필요 - skill-rules.json을 자동으로 읽음

---

### post-tool-use-tracker (PostToolUse)

**목적:** 세션 간 컨텍스트를 유지하기 위해 파일 변경 사항 추적

**파일:** `post-tool-use-tracker.sh`

**작동 방식:**

```
[Edit/Write/MultiEdit 완료] → [Hook 실행] → [repo 감지] → [캐시에 기록]
```

1. stdin으로 tool 정보 수신 (tool_name, file_path, session_id)
2. Edit, MultiEdit, Write 도구만 처리 (다른 도구는 스킵)
3. Markdown 파일(.md, .markdown)은 스킵
4. 파일 경로에서 repo 자동 감지:
   - frontend, client, web, app, ui → frontend 계열
   - backend, server, api, src, services → backend 계열
   - database, prisma, migrations → database 계열
   - packages/* → 모노레포 패키지
   - examples/* → 예제 프로젝트
5. 세션별 캐시 디렉토리 생성: `$CLAUDE_PROJECT_DIR/.claude/tsc-cache/{session_id}/`
6. 다음 정보 기록:
   - `edited-files.log`: 타임스탬프, 파일 경로, repo 정보
   - `affected-repos.txt`: 변경된 repo 목록 (중복 제거)
   - `commands.txt`: repo별 빌드/TSC 명령어

**빌드 명령어 자동 감지:**
- package.json의 "build" 스크립트 확인
- 패키지 매니저 자동 감지 (pnpm > npm > yarn)
- Prisma 프로젝트: `npx prisma generate`
- TypeScript 프로젝트: tsconfig.app.json 또는 tsconfig.json 사용

**필수인 이유:** Claude가 코드베이스의 어떤 부분이 활성 상태인지 이해하는 데 도움을 줍니다.

**통합 방법:**
```bash
# 파일 복사
cp post-tool-use-tracker.sh your-project/.claude/hooks/

# 실행 권한 부여
chmod +x your-project/.claude/hooks/post-tool-use-tracker.sh
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

**커스터마이징:** ✅ 불필요 - 구조를 자동 감지

---

## 선택적 Hooks (커스터마이징 필요)

### tsc-check (PostToolUse)

**목적:** 파일 수정 후 즉시 TypeScript 컴파일 검사 실행

**파일:** `tsc-check.sh`

**작동 방식:**

```
[Edit/Write/MultiEdit 완료] → [TS/JS 파일 확인] → [repo별 TSC 실행] → [에러 시 경고]
```

1. stdin으로 tool 정보 수신
2. Write, Edit, MultiEdit 도구만 처리
3. .ts, .tsx, .js, .jsx 파일만 검사
4. 파일 경로에서 repo 감지 (하드코딩된 서비스 목록 사용)
5. repo별로 적절한 TSC 명령어 결정:
   - `tsconfig.app.json` 존재 시: `npx tsc --project tsconfig.app.json --noEmit`
   - `tsconfig.build.json` 존재 시: `npx tsc --project tsconfig.build.json --noEmit`
   - references 사용 시: `npx tsc --build --noEmit`
   - 기본: `npx tsc --noEmit`
6. TSC 명령어 캐시 (성능 최적화)
7. 에러 발견 시:
   - stderr로 즉시 표시
   - 캐시에 에러 정보 저장 (`last-errors.txt`, `affected-repos.txt`)
   - exit code 1 반환 (Claude에게 에러 알림)

**출력 예시 (에러 시):**
```
⚡ TypeScript check on: frontend backend
  Checking frontend... ❌ Errors found
  Checking backend... ✅ OK

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚨 TypeScript errors found in 1 repo(s):  frontend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

👉 IMPORTANT: Use the auto-error-resolver agent to fix the errors

WE DO NOT LEAVE A MESS BEHIND
Error Preview:
src/components/Button.tsx:15:3 - error TS2322: Type 'string' is not assignable...
... and 5 more errors
```

**⚠️ 경고:** 멀티 서비스 모노레포 구조용으로 설정됨

**통합 방법:**

**먼저 이 hook이 적합한지 확인:**
- ✅ 사용 적합: 멀티 서비스 TypeScript 모노레포
- ❌ 건너뛰기: 단일 서비스 프로젝트 또는 다른 빌드 설정

**사용하는 경우:**
1. tsc-check.sh 복사
2. **서비스 감지 부분 수정 (약 26번째 줄):**
   ```bash
   # 예시 서비스를 실제 서비스로 교체:
   case "$repo" in
       email|exports|form|frontend|projects|uploads|users|utilities|events|database)
       # ↑ 실제 서비스명으로 변경
   ```
3. settings.json에 추가하기 전에 수동으로 테스트

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
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/tsc-check.sh"
          }
        ]
      }
    ]
  }
}
```

**캐시 관리:**
- 캐시 위치: `~/.claude/tsc-cache/{session_id}/`
- 7일 이상 된 캐시 자동 삭제
- TSC 명령어 캐시로 반복 실행 성능 향상

**커스터마이징:** ⚠️⚠️⚠️ 많이 필요 - 서비스 목록을 프로젝트에 맞게 수정 필수

---

### trigger-build-resolver (Stop)

**목적:** 컴파일 실패 시 build-error-resolver agent를 자동 실행

**파일:** `trigger-build-resolver.sh`

**작동 방식:**

```
[사용자 Stop] → [서비스별 git status 확인] → [변경 있으면 claude agent 실행]
```

1. 정의된 서비스 디렉토리 목록 순회:
   ```bash
   services_dirs=("email" "exports" "form" "frontend" "projects" "uploads" "users" "utilities" "events" "database")
   ```
2. 각 서비스의 git 상태 확인 (개별 git repo 가정)
3. 변경사항이 있는 서비스 수집
4. 변경된 서비스가 있으면 Claude CLI로 agent 실행:
   ```bash
   claude --agent build-error-resolver <<EOF
   Build and fix errors in these specific services only: ${services_list}
   EOF
   ```

**디버그 로그:**
- 위치: `/tmp/claude-hook-debug.log`
- 모든 실행 정보 기록 (트리거 시간, 인자, 서비스 상태 등)

**의존성:**
- tsc-check hook이 정상 작동해야 함
- Claude CLI가 PATH에 있어야 함
- 각 서비스가 개별 git repository여야 함

**커스터마이징:** ⚠️ 서비스 목록 수정 필요

---

### error-handling-reminder (Stop)

**목적:** 세션 종료 시 에러 핸들링 모범 사례 셀프체크 리마인더 표시

**파일 구성:**
- `error-handling-reminder.sh` - npx tsx로 TypeScript 실행하는 wrapper
- `error-handling-reminder.ts` - 실제 로직 구현

**작동 방식:**

```
[사용자 Stop] → [편집된 파일 분석] → [코드 패턴 감지] → [맞춤 리마인더 출력]
```

1. stdin으로 hook input 수신
2. post-tool-use-tracker가 기록한 `edited-files.log` 로드
3. 각 파일을 카테고리별로 분류:
   - **backend**: controllers, services, routes, api, server 경로
   - **frontend**: components, features, client 경로
   - **database**: database, prisma, migrations 경로
4. 파일 내용 분석하여 패턴 감지:
   - `hasTryCatch`: try-catch 블록 존재 여부
   - `hasAsync`: async 함수 존재 여부
   - `hasPrisma`: Prisma 관련 코드 존재 여부
   - `hasController`: Controller 클래스 또는 라우터 정의 여부
   - `hasApiCall`: fetch, axios, apiClient 호출 여부
5. 위험 패턴이 감지된 경우에만 리마인더 출력

**출력 예시:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 ERROR HANDLING SELF-CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  Backend Changes Detected
   3 file(s) edited

   ❓ Did you add Sentry.captureException() in catch blocks?
   ❓ Are Prisma operations wrapped in error handling?
   ❓ Do controllers use BaseController.handleError()?

   💡 Backend Best Practice:
      - All errors should be captured to Sentry
      - Use appropriate error helpers for context
      - Controllers should extend BaseController

💡 Frontend Changes Detected
   2 file(s) edited

   ❓ Do API calls show user-friendly error messages?

   💡 Frontend Best Practice:
      - Use your notification system for user feedback
      - Error boundaries for component errors
      - Display user-friendly error messages

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 TIP: Disable with SKIP_ERROR_REMINDER=1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**스킵 조건:**
- `SKIP_ERROR_REMINDER=1` 환경변수 설정 시
- 테스트 파일 (*.test.ts, *.spec.ts)
- 설정 파일 (*.config.ts, *.d.ts)
- types/ 디렉토리 내 파일
- 스타일 파일 (*.styles.ts)

**커스터마이징:** ✅ 불필요 - 자동 분석

---

### stop-build-check-enhanced (Stop)

**목적:** 세션 종료 시 종합적인 TypeScript 빌드 검사 및 에러 해결 가이드

**파일:** `stop-build-check-enhanced.sh`

**작동 방식:**

```
[사용자 Stop] → [캐시된 repo 확인] → [TSC 실행] → [에러 집계] → [해결 가이드 출력]
```

1. session_id로 캐시 디렉토리 확인
2. `affected-repos.txt`에서 변경된 repo 목록 로드
3. `commands.txt`에서 각 repo의 TSC 명령어 로드
4. 각 repo에 대해 TSC 실행 및 에러 수집
5. 결과를 `results/` 디렉토리에 저장:
   - `{repo}-errors.txt`: repo별 에러 상세
   - `error-summary.txt`: repo별 에러 개수 요약
6. 에러 개수에 따른 다른 출력:

**5개 이상 에러 시:**
```
## TypeScript Build Errors Detected

Found 15 TypeScript errors across the following repos:
- frontend: 10 errors
- backend: 5 errors

Please use the auto-error-resolver agent to fix these errors systematically.
The error details have been cached for the resolver to use.

Run: Task(subagent_type='auto-error-resolver', ...)
```

**5개 미만 에러 시:**
```
## Minor TypeScript Errors

Found 3 TypeScript error(s). Here are the details:

  === Errors in frontend ===
  src/components/Button.tsx:15:3 - error TS2322: ...
  ...

Please fix these errors directly in the affected files.
```

**Exit 코드:**
- `0`: 에러 없음 (캐시 자동 삭제)
- `2`: 에러 있음 (Claude에게 피드백 전송)

**커스터마이징:** ⚠️ post-tool-use-tracker와 함께 사용 필요

---

## Hook 조합 권장 사항

### 기본 설정 (모든 프로젝트)
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [{
          "type": "command",
          "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [{
          "type": "command",
          "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
        }]
      }
    ]
  }
}
```

### TypeScript 모노레포 설정
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [{
          "type": "command",
          "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/tsc-check.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/error-handling-reminder.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Claude Code용 안내

**사용자를 위해 hooks를 설정할 때:**

1. **먼저 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)를 읽으세요**
2. **항상 두 개의 필수 hooks로 시작**
3. **Stop hooks 추가 전에 먼저 물어보세요** - 잘못 설정하면 차단될 수 있음
4. **설정 후 검증:**
   ```bash
   ls -la .claude/hooks/*.sh | grep rwx
   ```

**질문이 있으신가요?** [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)를 참조하세요
