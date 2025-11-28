# 공통 패턴 라이브러리

Skill 트리거를 위한 바로 사용 가능한 regex 및 glob 패턴입니다. 복사하여 커스터마이즈하세요.

---

## Intent 패턴 (Regex)

### 기능/엔드포인트 생성
```regex
(add|create|implement|build).*?(feature|endpoint|route|service|controller)
```

### 컴포넌트 생성
```regex
(create|add|make|build).*?(component|UI|page|modal|dialog|form)
```

### 데이터베이스 작업
```regex
(add|create|modify|update).*?(user|table|column|field|schema|migration)
(database|prisma).*?(change|update|query)
```

### 오류 처리
```regex
(fix|handle|catch|debug).*?(error|exception|bug)
(add|implement).*?(try|catch|error.*?handling)
```

### 설명 요청
```regex
(how does|how do|explain|what is|describe|tell me about).*?
```

### 워크플로우 작업
```regex
(create|add|modify|update).*?(workflow|step|branch|condition)
(debug|troubleshoot|fix).*?workflow
```

### 테스트
```regex
(write|create|add).*?(test|spec|unit.*?test)
```

---

## 파일 경로 패턴 (Glob)

### 프론트엔드
```glob
frontend/src/**/*.tsx        # 모든 React 컴포넌트
frontend/src/**/*.ts         # 모든 TypeScript 파일
frontend/src/components/**   # components 디렉토리만
```

### 백엔드 서비스
```glob
form/src/**/*.ts            # Form 서비스
email/src/**/*.ts           # Email 서비스
users/src/**/*.ts           # Users 서비스
projects/src/**/*.ts        # Projects 서비스
```

### 데이터베이스
```glob
**/schema.prisma            # Prisma 스키마 (어디서든)
**/migrations/**/*.sql      # 마이그레이션 파일
database/src/**/*.ts        # 데이터베이스 스크립트
```

### 워크플로우
```glob
form/src/workflow/**/*.ts              # 워크플로우 엔진
form/src/workflow-definitions/**/*.json # 워크플로우 정의
```

### 테스트 제외
```glob
**/*.test.ts                # TypeScript 테스트
**/*.test.tsx               # React 컴포넌트 테스트
**/*.spec.ts                # Spec 파일
```

---

## 콘텐츠 패턴 (Regex)

### Prisma/데이터베이스
```regex
import.*[Pp]risma                # Prisma import
PrismaService                    # PrismaService 사용
prisma\.                         # prisma.something
\.findMany\(                     # Prisma 쿼리 메서드
\.create\(
\.update\(
\.delete\(
```

### 컨트롤러/라우트
```regex
export class.*Controller         # Controller 클래스
router\.                         # Express router
app\.(get|post|put|delete|patch) # Express app 라우트
```

### 오류 처리
```regex
try\s*\{                        # Try 블록
catch\s*\(                      # Catch 블록
throw new                        # Throw 문
```

### React/컴포넌트
```regex
export.*React\.FC               # React 함수형 컴포넌트
export default function.*       # 기본 함수 export
useState|useEffect              # React hooks
```

---

**사용 예시:**

```json
{
  "my-skill": {
    "promptTriggers": {
      "intentPatterns": [
        "(create|add|build).*?(component|UI|page)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.tsx"
      ],
      "contentPatterns": [
        "export.*React\\.FC",
        "useState|useEffect"
      ]
    }
  }
}
```

---

**관련 파일:**
- [SKILL.md](SKILL.md) - 메인 skill 가이드
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 상세 트리거 문서
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 전체 스키마
