# 문제 해결 - Skill 활성화 문제

Skill 활성화 문제에 대한 종합 디버깅 가이드입니다.

## 목차

- [Skill이 트리거되지 않음](#skill이-트리거되지-않음)
  - [UserPromptSubmit이 제안하지 않음](#userpromptsubmit이-제안하지-않음)
  - [PreToolUse가 차단하지 않음](#pretooluse가-차단하지-않음)
- [오탐](#오탐)
- [Hook이 실행되지 않음](#hook이-실행되지-않음)
- [성능 문제](#성능-문제)

---

## Skill이 트리거되지 않음

### UserPromptSubmit이 제안하지 않음

**증상:** 질문을 했지만 출력에 skill 제안이 나타나지 않음.

**일반적인 원인:**

####  1. 키워드가 매칭되지 않음

**확인:**
- skill-rules.json의 `promptTriggers.keywords` 확인
- 프롬프트에 키워드가 실제로 있는지?
- 기억하세요: 대소문자 구분 없는 부분 문자열 매칭

**예시:**
```json
"keywords": ["layout", "grid"]
```
- "layout이 어떻게 작동하나요?" → ✅ "layout" 매칭
- "grid 시스템이 어떻게 작동하나요?" → ✅ "grid" 매칭
- "layouts가 어떻게 작동하나요?" → ✅ "layout" 매칭
- "어떻게 작동하나요?" → ❌ 매칭 안 됨

**해결:** skill-rules.json에 더 많은 키워드 변형 추가

#### 2. Intent 패턴이 너무 구체적

**확인:**
- `promptTriggers.intentPatterns` 확인
- https://regex101.com/에서 regex 테스트
- 더 넓은 패턴이 필요할 수 있음

**예시:**
```json
"intentPatterns": [
  "(create|add).*?(database.*?table)"  // 너무 구체적
]
```
- "database table 생성해줘" → ✅ 매칭
- "새 table 추가해줘" → ❌ 매칭 안 됨 ("database" 누락)

**해결:** 패턴을 넓히기:
```json
"intentPatterns": [
  "(create|add).*?(table|database)"  // 더 나음
]
```

#### 3. Skill 이름 오타

**확인:**
- SKILL.md frontmatter의 skill 이름
- skill-rules.json의 skill 이름
- 정확히 일치해야 함

**예시:**
```yaml
# SKILL.md
name: project-catalog-developer
```
```json
// skill-rules.json
"project-catalogue-developer": {  // ❌ 오타: catalogue vs catalog
  ...
}
```

**해결:** 이름이 정확히 일치하도록 수정

#### 4. JSON 구문 오류

**확인:**
```bash
cat .claude/skills/skill-rules.json | jq .
```

유효하지 않은 JSON이면 jq가 오류를 표시합니다.

**일반적인 오류:**
- 후행 쉼표
- 따옴표 누락
- 큰따옴표 대신 작은따옴표
- 문자열에서 이스케이프 안 된 문자

**해결:** JSON 구문 수정, jq로 검증

#### 디버그 명령

수동으로 hook 테스트:

```bash
echo '{"session_id":"debug","prompt":"여기에 테스트 프롬프트"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

예상: 출력에 skill이 나타나야 함.

---

### PreToolUse가 차단하지 않음

**증상:** guardrail을 트리거해야 할 파일을 편집했지만 차단이 발생하지 않음.

**일반적인 원인:**

#### 1. 파일 경로가 패턴과 매칭되지 않음

**확인:**
- 편집 중인 파일 경로
- skill-rules.json의 `fileTriggers.pathPatterns`
- Glob 패턴 문법

**예시:**
```json
"pathPatterns": [
  "frontend/src/**/*.tsx"
]
```
- 편집 중: `frontend/src/components/Dashboard.tsx` → ✅ 매칭
- 편집 중: `frontend/tests/Dashboard.test.tsx` → ✅ 매칭 (제외 추가 필요!)
- 편집 중: `backend/src/app.ts` → ❌ 매칭 안 됨

**해결:** glob 패턴 조정 또는 누락된 경로 추가

#### 2. pathExclusions로 제외됨

**확인:**
- 테스트 파일을 편집하고 있는지?
- `fileTriggers.pathExclusions` 확인

**예시:**
```json
"pathExclusions": [
  "**/*.test.ts",
  "**/*.spec.ts"
]
```
- 편집 중: `services/user.test.ts` → ❌ 제외됨
- 편집 중: `services/user.ts` → ✅ 제외 안 됨

**해결:** 테스트 제외가 너무 넓으면 좁히거나 제거

#### 3. 콘텐츠 패턴을 찾을 수 없음

**확인:**
- 파일에 실제로 패턴이 포함되어 있는지?
- `fileTriggers.contentPatterns` 확인
- Regex가 맞는지?

**예시:**
```json
"contentPatterns": [
  "import.*[Pp]risma"
]
```
- 파일 내용: `import { PrismaService } from './prisma'` → ✅ 매칭
- 파일 내용: `import { Database } from './db'` → ❌ 매칭 안 됨

**디버그:**
```bash
# 파일에 패턴이 있는지 확인
grep -i "prisma" path/to/file.ts
```

**해결:** 콘텐츠 패턴 조정 또는 누락된 import 추가

#### 4. 세션에서 이미 Skill 사용됨

**세션 상태 확인:**
```bash
ls .claude/hooks/state/
cat .claude/hooks/state/skills-used-{session-id}.json
```

**예시:**
```json
{
  "skills_used": ["database-verification"],
  "files_verified": []
}
```

skill이 `skills_used`에 있으면 이 세션에서 다시 차단하지 않음.

**해결:** 리셋하려면 상태 파일 삭제:
```bash
rm .claude/hooks/state/skills-used-{session-id}.json
```

#### 5. 파일 마커 존재

**파일에서 스킵 마커 확인:**
```bash
grep "@skip-validation" path/to/file.ts
```

발견되면 파일이 영구적으로 스킵됨.

**해결:** 검증이 다시 필요하면 마커 제거

#### 6. 환경 변수 재정의

**확인:**
```bash
echo $SKIP_DB_VERIFICATION
echo $SKIP_SKILL_GUARDRAILS
```

설정되어 있으면 skill이 비활성화됨.

**해결:** 환경 변수 해제:
```bash
unset SKIP_DB_VERIFICATION
```

#### 디버그 명령

수동으로 hook 테스트:

```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts 2>&1
{
  "session_id": "debug",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/root/git/your-project/form/src/services/user.ts"}
}
EOF
echo "Exit code: $?"
```

예상:
- 차단해야 하면 종료 코드 2 + stderr 메시지
- 허용해야 하면 종료 코드 0 + 출력 없음

---

## 오탐

**증상:** Skill이 트리거되지 않아야 할 때 트리거됨.

**일반적인 원인 및 해결책:**

### 1. 키워드가 너무 일반적

**문제:**
```json
"keywords": ["user", "system", "create"]  // 너무 넓음
```
- 트리거됨: "user manual", "file system", "create directory"

**해결:** 키워드를 더 구체적으로
```json
"keywords": [
  "user authentication",
  "user tracking",
  "create feature"
]
```

### 2. Intent 패턴이 너무 넓음

**문제:**
```json
"intentPatterns": [
  "(create)"  // "create"가 있는 모든 것에 매칭
]
```
- 트리거됨: "create file", "create folder", "create account"

**해결:** 패턴에 context 추가
```json
"intentPatterns": [
  "(create|add).*?(database|table|feature)"  // 더 구체적
]
```

**고급:** 부정 전방 탐색으로 제외
```regex
(create)(?!.*test).*?(feature)  // "test"가 나타나면 매칭 안 함
```

### 3. 파일 경로가 너무 일반적

**문제:**
```json
"pathPatterns": [
  "form/**"  // form/ 안의 모든 것에 매칭
]
```
- 트리거됨: 테스트 파일, 설정 파일, 모든 것

**해결:** 더 좁은 패턴 사용
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 서비스 파일만
  "form/src/controllers/**/*.ts"
]
```

### 4. 콘텐츠 패턴이 관련 없는 코드 감지

**문제:**
```json
"contentPatterns": [
  "Prisma"  // 주석, 문자열 등에서 매칭
]
```
- 트리거됨: `// 여기서 Prisma 사용하지 마세요`
- 트리거됨: `const note = "Prisma is cool"`

**해결:** 패턴을 더 구체적으로
```json
"contentPatterns": [
  "import.*[Pp]risma",        // import만
  "PrismaService\\.",         // 실제 사용만
  "prisma\\.(findMany|create)" // 특정 메서드
]
```

### 5. 적용 수준 조정

**최후의 수단:** 오탐이 빈번하면:

```json
{
  "enforcement": "block"  // "suggest"로 변경
}
```

차단 대신 권고로 변경됩니다.

---

## Hook이 실행되지 않음

**증상:** Hook이 전혀 실행되지 않음 - 제안도 차단도 없음.

**일반적인 원인:**

### 1. Hook이 등록되지 않음

**`.claude/settings.json` 확인:**
```bash
cat .claude/settings.json | jq '.hooks.UserPromptSubmit'
cat .claude/settings.json | jq '.hooks.PreToolUse'
```

예상: Hook 항목이 있어야 함

**해결:** 누락된 hook 등록 추가:
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

### 2. Bash 래퍼가 실행 가능하지 않음

**확인:**
```bash
ls -l .claude/hooks/*.sh
```

예상: `-rwxr-xr-x` (실행 가능)

**해결:**
```bash
chmod +x .claude/hooks/*.sh
```

### 3. 잘못된 Shebang

**확인:**
```bash
head -1 .claude/hooks/skill-activation-prompt.sh
```

예상: `#!/bin/bash`

**해결:** 첫 번째 줄에 올바른 shebang 추가

### 4. npx/tsx 사용 불가

**확인:**
```bash
npx tsx --version
```

예상: 버전 번호

**해결:** 의존성 설치:
```bash
cd .claude/hooks
npm install
```

### 5. TypeScript 컴파일 오류

**확인:**
```bash
cd .claude/hooks
npx tsc --noEmit skill-activation-prompt.ts
```

예상: 출력 없음 (오류 없음)

**해결:** TypeScript 구문 오류 수정

---

## 성능 문제

**증상:** Hooks가 느림, 프롬프트/편집 전 눈에 띄는 지연.

**일반적인 원인:**

### 1. 패턴이 너무 많음

**확인:**
- skill-rules.json의 패턴 수
- 각 패턴 = regex 컴파일 + 매칭

**해결:** 패턴 줄이기
- 유사한 패턴 결합
- 중복 패턴 제거
- 더 구체적인 패턴 사용 (더 빠른 매칭)

### 2. 복잡한 Regex

**문제:**
```regex
(create|add|modify|update|implement|build).*?(feature|endpoint|route|service|controller|component|UI|page)
```
- 긴 대안 = 느림

**해결:** 단순화
```regex
(create|add).*?(feature|endpoint)  // 더 적은 대안
```

### 3. 확인할 파일이 너무 많음

**문제:**
```json
"pathPatterns": [
  "**/*.ts"  // 모든 TypeScript 파일 확인
]
```

**해결:** 더 구체적으로
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 특정 디렉토리만
  "form/src/controllers/**/*.ts"
]
```

### 4. 큰 파일

콘텐츠 패턴 매칭은 전체 파일을 읽음 - 큰 파일의 경우 느림.

**해결:**
- 필요할 때만 콘텐츠 패턴 사용
- 파일 크기 제한 고려 (향후 개선 사항)

### 성능 측정

```bash
# UserPromptSubmit
time echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
time cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

**목표 지표:**
- UserPromptSubmit: < 100ms
- PreToolUse: < 200ms

---

**관련 파일:**
- [SKILL.md](SKILL.md) - 메인 skill 가이드
- [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md) - Hook 작동 방식
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 설정 참조
