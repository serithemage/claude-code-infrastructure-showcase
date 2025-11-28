---
name: skill-developer
description: Create and manage Claude Code skills following Anthropic best practices. Use when creating new skills, modifying skill-rules.json, understanding trigger patterns, working with hooks, debugging skill activation, or implementing progressive disclosure. Covers skill structure, YAML frontmatter, trigger types (keywords, intent patterns, file paths, content patterns), enforcement levels (block, suggest, warn), hook mechanisms (UserPromptSubmit, PreToolUse), session tracking, and the 500-line rule.
---

# Skill 개발자 가이드

## 목적

Anthropic의 공식 모범 사례(500-line rule, Progressive disclosure 패턴 포함)를 따르는 Claude Code의 skills 생성 및 관리에 대한 종합 가이드입니다.

## 이 Skill 사용 시점

다음을 언급할 때 자동 활성화됩니다:
- Skills 생성 또는 추가
- Skill triggers 또는 rules 수정
- Skill 활성화 작동 방식 이해
- Skill 활성화 문제 디버깅
- skill-rules.json 작업
- Hook 시스템 메커니즘
- Claude Code 모범 사례
- Progressive disclosure
- YAML frontmatter
- 500-line rule

---

## 시스템 개요

### 2-Hook 아키텍처

**1. UserPromptSubmit Hook** (사전 제안)
- **파일**: `.claude/hooks/skill-activation-prompt.ts`
- **트리거**: Claude가 사용자 프롬프트를 보기 전에
- **목적**: 키워드 + intent 패턴 기반으로 관련 skills 제안
- **방식**: 포맷된 리마인더를 context로 주입 (stdout → Claude 입력)
- **사용 사례**: 주제 기반 skills, 암시적 작업 감지

**2. Stop Hook - 오류 처리 리마인더** (부드러운 알림)
- **파일**: `.claude/hooks/error-handling-reminder.ts`
- **트리거**: Claude가 응답을 완료한 후에
- **목적**: 작성한 코드의 오류 처리에 대한 자가 평가를 위한 부드러운 리마인더
- **방식**: 편집된 파일에서 위험한 패턴을 분석하고, 필요시 리마인더 표시
- **사용 사례**: 차단 마찰 없이 오류 처리 인식 유지

**철학 변경 (2025-10-27):** Sentry/오류 처리를 위한 차단 PreToolUse에서 벗어났습니다. 대신, 워크플로우를 차단하지 않으면서 코드 품질 인식을 유지하는 부드러운 응답 후 리마인더를 사용합니다.

### 설정 파일

**위치**: `.claude/skills/skill-rules.json`

정의 내용:
- 모든 skills와 트리거 조건
- 적용 수준 (block, suggest, warn)
- 파일 경로 패턴 (glob)
- 콘텐츠 감지 패턴 (regex)
- 스킵 조건 (세션 추적, 파일 마커, 환경 변수)

---

## Skill 유형

### 1. Guardrail Skills

**목적:** 오류를 방지하는 핵심 모범 사례 강제

**특성:**
- Type: `"guardrail"`
- Enforcement: `"block"`
- Priority: `"critical"` 또는 `"high"`
- Skill 사용 전까지 파일 편집 차단
- 일반적인 실수 방지 (컬럼 이름, 치명적 오류)
- 세션 인식 (같은 세션에서 반복 알림 안 함)

**예시:**
- `database-verification` - Prisma 쿼리 전 테이블/컬럼 이름 검증
- `frontend-dev-guidelines` - React/TypeScript 패턴 강제

**사용 시점:**
- 런타임 오류를 유발하는 실수
- 데이터 무결성 관련 문제
- 치명적 호환성 문제

### 2. Domain Skills

**목적:** 특정 영역에 대한 종합 가이드 제공

**특성:**
- Type: `"domain"`
- Enforcement: `"suggest"`
- Priority: `"high"` 또는 `"medium"`
- 권고 사항, 강제 아님
- 주제 또는 도메인 특화
- 종합적인 문서화

**예시:**
- `backend-dev-guidelines` - Node.js/Express/TypeScript 패턴
- `frontend-dev-guidelines` - React/TypeScript 모범 사례
- `error-tracking` - Sentry 통합 가이드

**사용 시점:**
- 깊은 지식이 필요한 복잡한 시스템
- 모범 사례 문서화
- 아키텍처 패턴
- How-to 가이드

---

## 빠른 시작: 새 Skill 생성

### 1단계: Skill 파일 생성

**위치:** `.claude/skills/{skill-name}/SKILL.md`

**템플릿:**
```markdown
---
name: my-new-skill
description: 이 skill을 트리거하는 키워드를 포함한 간단한 설명. 주제, 파일 유형, 사용 사례를 언급하세요. 트리거 용어를 명시적으로 작성하세요.
---

# 내 새 Skill

## 목적
이 skill이 도움이 되는 부분

## 사용 시점
특정 시나리오와 조건

## 핵심 정보
실제 가이드, 문서, 패턴, 예시
```

**모범 사례:**
- ✅ **이름**: 소문자, 하이픈, 동명사 형태 선호
- ✅ **설명**: 모든 트리거 키워드/구문 포함 (최대 1024자)
- ✅ **내용**: 500줄 미만 - 상세 정보는 참조 파일 사용
- ✅ **예시**: 실제 코드 예시
- ✅ **구조**: 명확한 제목, 목록, 코드 블록

### 2단계: skill-rules.json에 추가

전체 스키마는 [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)를 참조하세요.

**기본 템플릿:**
```json
{
  "my-new-skill": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "medium",
    "promptTriggers": {
      "keywords": ["keyword1", "keyword2"],
      "intentPatterns": ["(create|add).*?something"]
    }
  }
}
```

### 3단계: 트리거 테스트

**UserPromptSubmit 테스트:**
```bash
echo '{"session_id":"test","prompt":"테스트 프롬프트"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**PreToolUse 테스트:**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

### 4단계: 패턴 개선

테스트 결과에 따라:
- 누락된 키워드 추가
- 오탐을 줄이기 위해 intent 패턴 개선
- 파일 경로 패턴 조정
- 실제 파일에 대해 콘텐츠 패턴 테스트

### 5단계: Anthropic 모범 사례 준수

✅ SKILL.md 500줄 미만 유지
✅ 참조 파일로 Progressive disclosure 사용
✅ 100줄 초과 참조 파일에 목차 추가
✅ 트리거 키워드가 포함된 상세 설명 작성
✅ 문서화 전 3개 이상의 실제 시나리오로 테스트
✅ 실제 사용 기반으로 반복 개선

---

## 적용 수준

### BLOCK (핵심 Guardrails)

- Edit/Write 도구 실행을 물리적으로 방지
- Hook에서 종료 코드 2, stderr → Claude
- Claude가 메시지를 보고 skill을 사용해야 진행 가능
- **사용 대상**: 치명적 실수, 데이터 무결성, 보안 문제

**예시:** 데이터베이스 컬럼 이름 검증

### SUGGEST (권장)

- Claude가 프롬프트를 보기 전에 리마인더 주입
- Claude가 관련 skills를 인식
- 강제 아님, 권고만
- **사용 대상**: 도메인 가이드, 모범 사례, how-to 가이드

**예시:** 프론트엔드 개발 가이드라인

### WARN (선택)

- 낮은 우선순위 제안
- 권고만, 최소 적용
- **사용 대상**: 있으면 좋은 제안, 정보성 리마인더

**드물게 사용** - 대부분의 skills는 BLOCK 또는 SUGGEST입니다.

---

## 스킵 조건 및 사용자 제어

### 1. 세션 추적

**목적:** 같은 세션에서 반복 알림 방지

**작동 방식:**
- 첫 번째 편집 → Hook이 차단하고 세션 상태 업데이트
- 두 번째 편집 (같은 세션) → Hook이 허용
- 다른 세션 → 다시 차단

**상태 파일:** `.claude/hooks/state/skills-used-{session_id}.json`

### 2. 파일 마커

**목적:** 검증된 파일의 영구 스킵

**마커:** `// @skip-validation`

**사용법:**
```typescript
// @skip-validation
import { PrismaService } from './prisma';
// 이 파일은 수동으로 검증되었습니다
```

**참고:** 남용 시 목적이 무력화되므로 신중하게 사용

### 3. 환경 변수

**목적:** 긴급 비활성화, 임시 재정의

**전역 비활성화:**
```bash
export SKIP_SKILL_GUARDRAILS=true  # 모든 PreToolUse 차단 비활성화
```

**Skill별:**
```bash
export SKIP_DB_VERIFICATION=true
export SKIP_ERROR_REMINDER=true
```

---

## 테스트 체크리스트

새 skill 생성 시 확인 사항:

- [ ] `.claude/skills/{name}/SKILL.md`에 skill 파일 생성됨
- [ ] 이름과 설명이 포함된 올바른 frontmatter
- [ ] `skill-rules.json`에 항목 추가됨
- [ ] 실제 프롬프트로 키워드 테스트됨
- [ ] 다양한 변형으로 intent 패턴 테스트됨
- [ ] 실제 파일로 파일 경로 패턴 테스트됨
- [ ] 파일 내용에 대해 콘텐츠 패턴 테스트됨
- [ ] 차단 메시지가 명확하고 실행 가능함 (guardrail인 경우)
- [ ] 스킵 조건이 적절히 설정됨
- [ ] 우선순위 수준이 중요도와 일치함
- [ ] 테스트에서 오탐 없음
- [ ] 테스트에서 미탐 없음
- [ ] 성능 허용 범위 (<100ms 또는 <200ms)
- [ ] JSON 구문 검증: `jq . skill-rules.json`
- [ ] **SKILL.md 500줄 미만** ⭐
- [ ] 필요시 참조 파일 생성됨
- [ ] 100줄 초과 파일에 목차 추가됨

---

## 참조 파일

특정 주제에 대한 상세 정보는 다음을 참조하세요:

### [TRIGGER_TYPES.md](TRIGGER_TYPES.md)
모든 트리거 유형 종합 가이드:
- 키워드 트리거 (명시적 주제 매칭)
- Intent 패턴 (암시적 동작 감지)
- 파일 경로 트리거 (glob 패턴)
- 콘텐츠 패턴 (파일 내 regex)
- 각각의 모범 사례와 예시
- 일반적인 함정과 테스트 전략

### [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)
skill-rules.json 전체 스키마:
- 전체 TypeScript 인터페이스 정의
- 필드별 설명
- 전체 guardrail skill 예시
- 전체 domain skill 예시
- 검증 가이드 및 일반적인 오류

### [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md)
Hook 내부 심층 분석:
- UserPromptSubmit 흐름 (상세)
- PreToolUse 흐름 (상세)
- 종료 코드 동작 표 (핵심)
- 세션 상태 관리
- 성능 고려사항

### [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
종합 디버깅 가이드:
- Skill이 트리거되지 않음 (UserPromptSubmit)
- PreToolUse가 차단하지 않음
- 오탐 (너무 많은 트리거)
- Hook이 전혀 실행되지 않음
- 성능 문제

### [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md)
바로 사용 가능한 패턴 모음:
- Intent 패턴 라이브러리 (regex)
- 파일 경로 패턴 라이브러리 (glob)
- 콘텐츠 패턴 라이브러리 (regex)
- 사용 사례별 정리
- 복사-붙여넣기 가능

### [ADVANCED.md](ADVANCED.md)
향후 개선 사항 및 아이디어:
- 동적 규칙 업데이트
- Skill 종속성
- 조건부 적용
- Skill 분석
- Skill 버전 관리

---

## 빠른 참조 요약

### 새 Skill 생성 (5단계)

1. frontmatter와 함께 `.claude/skills/{name}/SKILL.md` 생성
2. `.claude/skills/skill-rules.json`에 항목 추가
3. `npx tsx` 명령으로 테스트
4. 테스트 기반으로 패턴 개선
5. SKILL.md 500줄 미만 유지

### 트리거 유형

- **Keywords**: 명시적 주제 언급
- **Intent**: 암시적 동작 감지
- **File Paths**: 위치 기반 활성화
- **Content**: 기술 특화 감지

상세 내용은 [TRIGGER_TYPES.md](TRIGGER_TYPES.md)를 참조하세요.

### 적용

- **BLOCK**: 종료 코드 2, 핵심 사항만
- **SUGGEST**: context 주입, 가장 일반적
- **WARN**: 권고, 드물게 사용

### 스킵 조건

- **세션 추적**: 자동 (반복 알림 방지)
- **파일 마커**: `// @skip-validation` (영구 스킵)
- **환경 변수**: `SKIP_SKILL_GUARDRAILS` (긴급 비활성화)

### Anthropic 모범 사례

✅ **500-line rule**: SKILL.md 500줄 미만 유지
✅ **Progressive disclosure**: 상세 정보는 참조 파일 사용
✅ **목차**: 100줄 초과 참조 파일에 추가
✅ **한 단계 깊이**: 참조를 깊게 중첩하지 않음
✅ **풍부한 설명**: 모든 트리거 키워드 포함 (최대 1024자)
✅ **먼저 테스트**: 광범위한 문서화 전 3개 이상의 평가 구축
✅ **동명사 명명**: 동사 + -ing 선호 (예: "processing-pdfs")

### 문제 해결

수동으로 hooks 테스트:
```bash
# UserPromptSubmit
echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

전체 디버깅 가이드는 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)를 참조하세요.

---

## 관련 파일

**설정:**
- `.claude/skills/skill-rules.json` - 마스터 설정
- `.claude/hooks/state/` - 세션 추적
- `.claude/settings.json` - Hook 등록

**Hooks:**
- `.claude/hooks/skill-activation-prompt.ts` - UserPromptSubmit
- `.claude/hooks/error-handling-reminder.ts` - Stop 이벤트 (부드러운 리마인더)

**모든 Skills:**
- `.claude/skills/*/SKILL.md` - Skill 콘텐츠 파일

---

**Skill 상태**: COMPLETE - Anthropic 모범 사례에 따라 재구성됨 ✅
**줄 수**: < 500 (500-line rule 준수) ✅
**Progressive Disclosure**: 상세 정보를 위한 참조 파일 ✅

**다음**: 더 많은 skills 생성, 사용 기반 패턴 개선
