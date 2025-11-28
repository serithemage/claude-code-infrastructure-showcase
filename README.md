# Claude Code Infrastructure Showcase

**실제 프로덕션에서 검증된 Claude Code infrastructure의 레퍼런스 라이브러리입니다.**

복잡한 TypeScript microservices 프로젝트를 6개월간 실제로 운영하며 얻은 경험을 바탕으로, 이 showcase는 "skills가 자동으로 활성화되지 않는" 문제를 해결하고 Claude Code를 enterprise 개발에 맞게 확장한 패턴과 시스템을 제공합니다.

> 역자주: 이 레포지토리는 원 저자가 Reddit에 포스팅한 글 ["Claude Code는 괴물이다 – 6개월 하드코어 사용에서 얻은 팁들"](https://rosettalens.com/s/ko/claude-code-is-a-beast-tips-from-6-months-of-hardcore-use)을 읽은 사람들로부터 수백 건의 요청 후, 커뮤니티가 이 패턴들을 구현할 수 있도록 이 showcase가 만들어졌습니다. 아직 해당 포스트를 읽지 않으셨다면, 먼저 포스트를 읽고 오시는 것을 권합니다.

> **이것은 동작하는 애플리케이션이 아닙니다** - 레퍼런스 라이브러리입니다. 필요한 부분을 자신의 프로젝트에 복사해서 사용하세요.

---

## 포함된 내용

**프로덕션에서 검증된 infrastructure:**
- ✅ **자동 활성화 skills** (hooks 사용)
- ✅ **모듈형 skill 패턴** (progressive disclosure를 활용한 500줄 규칙)
- ✅ **복잡한 작업을 위한 전문 agents**
- ✅ **context 리셋에도 살아남는 dev docs 시스템**
- ✅ **범용 블로그 도메인을 사용한 포괄적인 예제**

**구축에 투자된 시간:** 6개월의 반복 개발
**프로젝트에 통합하는 데 걸리는 시간:** 15-30분

---

## 빠른 시작 - 경로 선택하기

### 🤖 Claude Code를 사용해서 통합하시나요?

**Claude:** AI 지원 설정을 위한 단계별 통합 가이드는 [`CLAUDE_INTEGRATION_GUIDE.md`](CLAUDE_INTEGRATION_GUIDE.md)를 읽어보세요.

### 🎯 skill 자동 활성화가 필요해요

**핵심 기능:** 정말로 필요할 때 자동으로 활성화되는 skills.

**필요한 것:**
1. skill-activation hooks (파일 2개)
2. 작업에 관련된 skill 1~2개
3. 15분

**👉 [설정 가이드: .claude/hooks/README.md](.claude/hooks/README.md)**

### 📚 skill 하나만 추가하고 싶어요

[skills catalog](.claude/skills/)를 둘러보고 필요한 것을 복사하세요.

**사용 가능한 skills:**
- **backend-dev-guidelines** - Node.js/Express/TypeScript 패턴
- **frontend-dev-guidelines** - React/TypeScript/MUI v7 패턴
- **skill-developer** - skill 생성을 위한 Meta-skill
- **route-tester** - 인증된 API route 테스트
- **error-tracking** - Sentry 통합 패턴

**👉 [Skills 가이드: .claude/skills/README.md](.claude/skills/README.md)**

### 🤖 전문 agents가 필요해요

복잡한 작업을 위한 프로덕션 검증된 10개의 agents:
- Code architecture 리뷰
- Refactoring 지원
- Documentation 생성
- Error 디버깅
- 그 외 다수...

**👉 [Agents 가이드: .claude/agents/README.md](.claude/agents/README.md)**

---

## 무엇이 다른가요?

### 자동 활성화 혁신

**문제:** Claude Code skills는 그냥 거기 있기만 합니다. 사용하려면 기억해야 합니다.

**해결책:** UserPromptSubmit hook이:
- 프롬프트를 분석하고
- 파일 context를 확인하며
- 관련 skills를 자동으로 제안하고
- `skill-rules.json` 설정을 통해 작동합니다

**결과:** Skills는 기억할 때가 아니라 필요할 때 활성화됩니다.

### 프로덕션에서 검증된 패턴

이것들은 이론적 예제가 아닙니다 - 다음에서 추출된 실제 패턴입니다:
- ✅ 프로덕션 환경의 6개 microservices
- ✅ 50,000줄 이상의 TypeScript 코드
- ✅ 복잡한 data grid를 가진 React frontend
- ✅ 정교한 workflow engine
- ✅ 6개월간 매일 사용한 Claude Code

이 패턴들이 동작하는 이유는 실제 문제를 해결했기 때문입니다.

### 모듈형 Skills (500줄 규칙)

큰 skills는 context 제한에 걸립니다. 해결책:

```
skill-name/
  SKILL.md                  # <500줄, 상위 레벨 가이드
  resources/
    topic-1.md              # 각각 <500줄
    topic-2.md
    topic-3.md
```

**Progressive disclosure:** Claude가 먼저 메인 skill을 로드하고, 필요할 때만 리소스를 로드합니다.

---

## Repository 구조

```
.claude/
├── skills/                 # 프로덕션 skills 5개
│   ├── backend-dev-guidelines/  (리소스 파일 12개)
│   ├── frontend-dev-guidelines/ (리소스 파일 11개)
│   ├── skill-developer/         (리소스 파일 7개)
│   ├── route-tester/
│   ├── error-tracking/
│   └── skill-rules.json    # Skill 활성화 설정
├── hooks/                  # 자동화를 위한 hooks 6개
│   ├── skill-activation-prompt.*  (필수)
│   ├── post-tool-use-tracker.sh   (필수)
│   ├── tsc-check.sh        (선택, 커스터마이징 필요)
│   └── trigger-build-resolver.sh  (선택)
├── agents/                 # 전문 agents 10개
│   ├── code-architecture-reviewer.md
│   ├── refactor-planner.md
│   ├── frontend-error-fixer.md
│   └── ... 7개 더
└── commands/               # slash commands 3개
    ├── dev-docs.md
    └── ...

dev/
└── active/                 # Dev docs 패턴 예제
    └── public-infrastructure-repo/
```

---

## 컴포넌트 카탈로그

### 🎨 Skills (5개)

| Skill | 줄 수 | 목적 | 최적 사용처 |
|-------|-------|---------|----------|
| [**skill-developer**](.claude/skills/skill-developer/) | 426 | skills 생성 및 관리 | Meta-개발 |
| [**backend-dev-guidelines**](.claude/skills/backend-dev-guidelines/) | 304 | Express/Prisma/Sentry 패턴 | Backend APIs |
| [**frontend-dev-guidelines**](.claude/skills/frontend-dev-guidelines/) | 398 | React/MUI v7/TypeScript | React frontends |
| [**route-tester**](.claude/skills/route-tester/) | 389 | 인증된 routes 테스트 | API 테스팅 |
| [**error-tracking**](.claude/skills/error-tracking/) | ~250 | Sentry 통합 | Error 모니터링 |

**모든 skills는 모듈형 패턴을 따릅니다** - 메인 파일 + progressive disclosure를 위한 리소스 파일.

**👉 [skills 통합 방법 →](.claude/skills/README.md)**

### 🪝 Hooks (6개)

| Hook | 타입 | 필수? | 커스터마이징 |
|------|------|-----------|---------------|
| skill-activation-prompt | UserPromptSubmit | ✅ 필수 | ✅ 불필요 |
| post-tool-use-tracker | PostToolUse | ✅ 필수 | ✅ 불필요 |
| tsc-check | Stop | ⚠️ 선택 | ⚠️ 무거움 - monorepo 전용 |
| trigger-build-resolver | Stop | ⚠️ 선택 | ⚠️ 무거움 - monorepo 전용 |
| error-handling-reminder | Stop | ⚠️ 선택 | ⚠️ 보통 |
| stop-build-check-enhanced | Stop | ⚠️ 선택 | ⚠️ 보통 |

**필수 hooks 2개부터 시작하세요** - skill 자동 활성화를 가능하게 하며 바로 사용 가능합니다.

**👉 [Hook 설정 가이드 →](.claude/hooks/README.md)**

### 🤖 Agents (10개)

**독립형 - 복사해서 바로 사용!**

| Agent | 목적 |
|-------|---------|
| code-architecture-reviewer | 아키텍처 일관성을 위한 코드 리뷰 |
| code-refactor-master | Refactoring 계획 및 실행 |
| documentation-architect | 포괄적인 documentation 생성 |
| frontend-error-fixer | Frontend 에러 디버깅 |
| plan-reviewer | 개발 계획 리뷰 |
| refactor-planner | Refactoring 전략 수립 |
| web-research-specialist | 온라인 기술 이슈 리서치 |
| auth-route-tester | 인증된 endpoints 테스트 |
| auth-route-debugger | Auth 이슈 디버깅 |
| auto-error-resolver | TypeScript 에러 자동 수정 |

**👉 [Agents 작동 방식 →](.claude/agents/README.md)**

### 💬 Slash Commands (3개)

| Command | 목적 |
|---------|---------|
| /dev-docs | 구조화된 dev documentation 생성 |
| /dev-docs-update | context 리셋 전에 문서 업데이트 |
| /route-research-for-testing | 테스팅을 위한 route 패턴 리서치 |

---

## 핵심 개념

### Hooks + skill-rules.json = 자동 활성화

**시스템:**
1. **skill-activation-prompt hook**이 모든 사용자 프롬프트에서 실행됨
2. **skill-rules.json**에서 트리거 패턴을 확인
3. 관련 skills를 자동으로 제안
4. Skills는 필요할 때만 로드됨

**이것은 Claude Code skills의 최대 문제를 해결합니다**: 스스로 활성화되지 않는다는 점.

### Progressive Disclosure (500줄 규칙)

**문제:** 큰 skills는 context 제한에 걸립니다

**해결책:** 모듈형 구조
- 메인 SKILL.md <500줄 (개요 + 네비게이션)
- 리소스 파일 각각 <500줄 (심화 내용)
- Claude가 필요에 따라 점진적으로 로드

**예시:** backend-dev-guidelines는 routing, controllers, services, repositories, testing 등을 다루는 12개의 리소스 파일을 가지고 있습니다.

### Dev Docs 패턴

**문제:** Context 리셋이 프로젝트 context를 잃어버립니다

**해결책:** 3-파일 구조
- `[task]-plan.md` - 전략적 계획
- `[task]-context.md` - 주요 결정사항과 파일들
- `[task]-tasks.md` - 체크리스트 형식

**연계:** `/dev-docs` slash command로 이것들을 자동 생성

---

## ⚠️ 중요: 그대로 동작하지 않는 것들

### settings.json
포함된 `settings.json`은 **예제일 뿐**입니다:
- Stop hooks는 특정 monorepo 구조를 참조합니다
- Service 이름들(blog-api 등)은 예제입니다
- MCP servers가 귀하의 설정에 존재하지 않을 수 있습니다

**사용하려면:**
1. UserPromptSubmit과 PostToolUse hooks만 추출하세요
2. Stop hooks는 커스터마이징하거나 건너뛰세요
3. 귀하의 설정에 맞게 MCP server 목록을 업데이트하세요

### 블로그 도메인 예제
Skills는 범용 블로그 예제를 사용합니다 (Post/Comment/User):
- 이것들은 **학습용 예제**이지 요구사항이 아닙니다
- 패턴은 모든 도메인(e-commerce, SaaS 등)에서 동작합니다
- 패턴을 귀하의 비즈니스 로직에 맞게 조정하세요

### Hook 디렉토리 구조
일부 hooks는 특정 구조를 기대합니다:
- `tsc-check.sh`는 service 디렉토리들을 기대합니다
- 귀하의 프로젝트 레이아웃에 맞게 커스터마이징하세요

---

## 통합 워크플로우

**권장 접근법:**

### Phase 1: Skill 활성화 (15분)
1. skill-activation-prompt hook 복사
2. post-tool-use-tracker hook 복사
3. settings.json 업데이트
4. hook dependencies 설치

### Phase 2: 첫 번째 Skill 추가 (10분)
1. 관련 있는 skill 하나 선택
2. skill 디렉토리 복사
3. skill-rules.json 생성/업데이트
4. path 패턴 커스터마이징

### Phase 3: 테스트 & 반복 (5분)
1. 파일 편집 - skill이 활성화되어야 함
2. 질문하기 - skill이 제안되어야 함
3. 필요에 따라 더 많은 skills 추가

### Phase 4: 선택적 개선사항
- 유용한 agents 추가
- Slash commands 추가
- Stop hooks 커스터마이징 (고급)

---

## 도움 받기

### 사용자를 위해
**통합에 문제가 있나요?**
1. [CLAUDE_INTEGRATION_GUIDE.md](CLAUDE_INTEGRATION_GUIDE.md)를 확인하세요
2. Claude에게 물어보세요: "왜 [skill]이 활성화되지 않나요?"
3. 프로젝트 구조와 함께 이슈를 등록하세요

### Claude Code를 위해
사용자의 통합을 도울 때:
1. **먼저 CLAUDE_INTEGRATION_GUIDE.md를 읽으세요**
2. 프로젝트 구조에 대해 물어보세요
3. 무작정 복사하지 말고 커스터마이징하세요
4. 통합 후 검증하세요

---

## 해결하는 문제들

### 이 Infrastructure 이전

❌ Skills가 자동으로 활성화되지 않습니다
❌ 어떤 skill을 사용할지 기억해야 합니다
❌ 큰 skills가 context 제한에 걸립니다
❌ Context 리셋이 프로젝트 지식을 잃어버립니다
❌ 개발 전반에 일관성이 없습니다
❌ 매번 수동으로 agent를 호출해야 합니다

### 이 Infrastructure 이후

✅ Skills가 context를 기반으로 스스로 제안합니다
✅ Hooks가 적절한 시점에 skills를 트리거합니다
✅ 모듈형 skills가 context 제한 내에 머물러 있습니다
✅ Dev docs가 리셋 간에 지식을 보존합니다
✅ Guardrails를 통한 일관된 패턴을 제공합니다
✅ Agents가 복잡한 작업을 간소화합니다

---

## 커뮤니티

**유용했나요?**

- ⭐ 이 repo에 Star를 눌러주세요
- 🐛 이슈를 보고하거나 개선사항을 제안해주세요
- 💬 자신만의 skills/hooks/agents를 공유해주세요
- 📝 귀하의 도메인에서 나온 예제를 기여해주세요

**배경:**
이 infrastructure는 제가 Reddit에 올린 글 ["Claude Code is a Beast – Tips from 6 Months of Hardcore Use"](https://www.reddit.com/r/ClaudeAI/comments/1oivjvm/claude_code_is_a_beast_tips_from_6_months_of/)에서 자세히 설명되었습니다. 수백 건의 요청 후, 커뮤니티가 이 패턴들을 구현할 수 있도록 이 showcase가 만들어졌습니다.


---

## 라이선스

MIT License - 상업적 또는 개인적 프로젝트에서 자유롭게 사용하세요.

---

## 빠른 링크

- 📖 [Claude Integration Guide](CLAUDE_INTEGRATION_GUIDE.md) - AI 지원 설정을 위해
- 🎨 [Skills Documentation](.claude/skills/README.md)
- 🪝 [Hooks Setup](.claude/hooks/README.md)
- 🤖 [Agents Guide](.claude/agents/README.md)
- 📝 [Dev Docs Pattern](dev/README.md)

**여기서 시작하세요:** 필수 hooks 2개를 복사하고, skill 하나를 추가한 다음, 자동 활성화의 마법을 경험하세요.
