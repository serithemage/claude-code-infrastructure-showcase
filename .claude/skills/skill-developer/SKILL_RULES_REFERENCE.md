# skill-rules.json - 전체 참조

`.claude/skills/skill-rules.json`에 대한 전체 스키마 및 설정 참조입니다.

## 목차

- [파일 위치](#파일-위치)
- [전체 TypeScript 스키마](#전체-typescript-스키마)
- [필드 가이드](#필드-가이드)
- [예시: Guardrail Skill](#예시-guardrail-skill)
- [예시: Domain Skill](#예시-domain-skill)
- [검증](#검증)

---

## 파일 위치

**경로:** `.claude/skills/skill-rules.json`

이 JSON 파일은 자동 활성화 시스템을 위한 모든 skills와 트리거 조건을 정의합니다.

---

## 전체 TypeScript 스키마

```typescript
interface SkillRules {
    version: string;
    skills: Record<string, SkillRule>;
}

interface SkillRule {
    type: 'guardrail' | 'domain';
    enforcement: 'block' | 'suggest' | 'warn';
    priority: 'critical' | 'high' | 'medium' | 'low';

    promptTriggers?: {
        keywords?: string[];
        intentPatterns?: string[];  // Regex 문자열
    };

    fileTriggers?: {
        pathPatterns: string[];     // Glob 패턴
        pathExclusions?: string[];  // Glob 패턴
        contentPatterns?: string[]; // Regex 문자열
        createOnly?: boolean;       // 파일 생성 시에만 트리거
    };

    blockMessage?: string;  // Guardrails용, {file_path} 플레이스홀더

    skipConditions?: {
        sessionSkillUsed?: boolean;      // 세션에서 사용된 경우 스킵
        fileMarkers?: string[];          // 예: ["@skip-validation"]
        envOverride?: string;            // 예: "SKIP_DB_VERIFICATION"
    };
}
```

---

## 필드 가이드

### 최상위 레벨

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `version` | string | 예 | 스키마 버전 (현재 "1.0") |
| `skills` | object | 예 | skill 이름 → SkillRule 맵 |

### SkillRule 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `type` | string | 예 | "guardrail" (강제) 또는 "domain" (권고) |
| `enforcement` | string | 예 | "block" (PreToolUse), "suggest" (UserPromptSubmit), 또는 "warn" |
| `priority` | string | 예 | "critical", "high", "medium", 또는 "low" |
| `promptTriggers` | object | 선택 | UserPromptSubmit hook용 트리거 |
| `fileTriggers` | object | 선택 | PreToolUse hook용 트리거 |
| `blockMessage` | string | 선택* | enforcement="block"인 경우 필수. `{file_path}` 플레이스홀더 사용 |
| `skipConditions` | object | 선택 | 탈출 조건 및 세션 추적 |

*Guardrails에 필수

### promptTriggers 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `keywords` | string[] | 선택 | 정확한 부분 문자열 매칭 (대소문자 구분 안 함) |
| `intentPatterns` | string[] | 선택 | Intent 감지용 Regex 패턴 |

### fileTriggers 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `pathPatterns` | string[] | 예* | 파일 경로용 Glob 패턴 |
| `pathExclusions` | string[] | 선택 | 제외할 Glob 패턴 (예: 테스트 파일) |
| `contentPatterns` | string[] | 선택 | 파일 내용 매칭용 Regex 패턴 |
| `createOnly` | boolean | 선택 | 새 파일 생성 시에만 트리거 |

*fileTriggers가 있는 경우 필수

### skipConditions 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `sessionSkillUsed` | boolean | 선택 | 이 세션에서 skill이 이미 사용된 경우 스킵 |
| `fileMarkers` | string[] | 선택 | 파일에 주석 마커가 포함된 경우 스킵 |
| `envOverride` | string | 선택 | Skill을 비활성화하는 환경 변수 이름 |

---

## 예시: Guardrail Skill

모든 기능이 포함된 차단 guardrail skill의 전체 예시:

```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",

    "promptTriggers": {
      "keywords": [
        "prisma",
        "database",
        "table",
        "column",
        "schema",
        "query",
        "migration"
      ],
      "intentPatterns": [
        "(add|create|implement).*?(user|login|auth|tracking|feature)",
        "(modify|update|change).*?(table|column|schema|field)",
        "database.*?(change|update|modify|migration)"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "**/schema.prisma",
        "**/migrations/**/*.sql",
        "database/src/**/*.ts",
        "form/src/**/*.ts",
        "email/src/**/*.ts",
        "users/src/**/*.ts",
        "projects/src/**/*.ts",
        "utilities/src/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts",
        "**/*.spec.ts"
      ],
      "contentPatterns": [
        "import.*[Pp]risma",
        "PrismaService",
        "prisma\\.",
        "\\.findMany\\(",
        "\\.findUnique\\(",
        "\\.findFirst\\(",
        "\\.create\\(",
        "\\.createMany\\(",
        "\\.update\\(",
        "\\.updateMany\\(",
        "\\.upsert\\(",
        "\\.delete\\(",
        "\\.deleteMany\\("
      ]
    },

    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED ACTION:\n1. Use Skill tool: 'database-verification'\n2. Verify ALL table and column names against schema\n3. Check database structure with DESCRIBE commands\n4. Then retry this edit\n\nReason: Prevent column name errors in Prisma queries\nFile: {file_path}\n\n💡 TIP: Add '// @skip-validation' comment to skip future checks",

    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": [
        "@skip-validation"
      ],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

### Guardrails 핵심 포인트

1. **type**: "guardrail"이어야 함
2. **enforcement**: "block"이어야 함
3. **priority**: 보통 "critical" 또는 "high"
4. **blockMessage**: 필수, 명확하고 실행 가능한 단계
5. **skipConditions**: 세션 추적으로 반복 알림 방지
6. **fileTriggers**: 보통 경로와 콘텐츠 패턴 모두 포함
7. **contentPatterns**: 기술의 실제 사용 감지

---

## 예시: Domain Skill

제안 기반 domain skill의 전체 예시:

```json
{
  "project-catalog-developer": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",

    "promptTriggers": {
      "keywords": [
        "layout",
        "layout system",
        "grid",
        "grid layout",
        "toolbar",
        "column",
        "cell editor",
        "cell renderer",
        "submission",
        "submissions",
        "blog dashboard",
        "datagrid",
        "data grid",
        "CustomToolbar",
        "GridLayoutDialog",
        "useGridLayout",
        "auto-save",
        "column order",
        "column width",
        "filter",
        "sort"
      ],
      "intentPatterns": [
        "(how does|how do|explain|what is|describe).*?(layout|grid|toolbar|column|submission|catalog)",
        "(add|create|modify|change).*?(toolbar|column|cell|editor|renderer)",
        "blog dashboard.*?"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/features/submissions/**/*.tsx",
        "frontend/src/features/submissions/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.tsx",
        "**/*.test.ts"
      ]
    }
  }
}
```

### Domain Skills 핵심 포인트

1. **type**: "domain"이어야 함
2. **enforcement**: 보통 "suggest"
3. **priority**: "high" 또는 "medium"
4. **blockMessage**: 필요 없음 (차단하지 않음)
5. **skipConditions**: 선택 (덜 중요함)
6. **promptTriggers**: 보통 광범위한 키워드 포함
7. **fileTriggers**: 경로 패턴만 있을 수 있음 (콘텐츠는 덜 중요)

---

## 검증

### JSON 구문 확인

```bash
cat .claude/skills/skill-rules.json | jq .
```

유효한 경우 jq가 JSON을 예쁘게 출력합니다. 유효하지 않은 경우 오류를 표시합니다.

### 일반적인 JSON 오류

**후행 쉼표:**
```json
{
  "keywords": ["one", "two",]  // ❌ 후행 쉼표
}
```

**따옴표 누락:**
```json
{
  type: "guardrail"  // ❌ 키에 따옴표 누락
}
```

**작은따옴표 (유효하지 않은 JSON):**
```json
{
  'type': 'guardrail'  // ❌ 큰따옴표 사용해야 함
}
```

### 검증 체크리스트

- [ ] JSON 구문 유효함 (`jq` 사용)
- [ ] 모든 skill 이름이 SKILL.md 파일명과 일치함
- [ ] Guardrails에 `blockMessage` 있음
- [ ] 차단 메시지가 `{file_path}` 플레이스홀더 사용함
- [ ] Intent 패턴이 유효한 regex임 (regex101.com에서 테스트)
- [ ] 파일 경로 패턴이 올바른 glob 문법 사용함
- [ ] 콘텐츠 패턴이 특수 문자 이스케이프함
- [ ] 우선순위가 적용 수준과 일치함
- [ ] 중복 skill 이름 없음

---

**관련 파일:**
- [SKILL.md](SKILL.md) - 메인 skill 가이드
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 전체 트리거 문서
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - 설정 문제 디버깅
