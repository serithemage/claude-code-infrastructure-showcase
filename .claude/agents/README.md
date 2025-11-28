# Agents

복잡한 다단계 작업을 위한 특화된 에이전트.

---

## Agents란?

Agents는 특정 복잡한 작업을 처리하는 자율적인 Claude 인스턴스입니다. Skills(인라인 가이드를 제공하는)와 달리 agents는:
- 별도의 하위 작업으로 실행
- 최소한의 감독으로 자율적으로 작업
- 특화된 도구 접근 권한 보유
- 완료 시 종합 보고서 반환

**핵심 장점:** Agents는 **독립적** - `.md` 파일만 복사하면 즉시 사용 가능!

---

## 사용 가능한 Agents (10개)

### code-architecture-reviewer
**목적:** 아키텍처 일관성 및 모범 사례에 대한 코드 리뷰

**사용 시기:**
- 새 기능 구현 후
- 중요한 변경사항 병합 전
- 코드 리팩토링 시
- 아키텍처 결정 검증 시

**통합:** ✅ 그대로 복사

---

### code-refactor-master
**목적:** 종합적인 리팩토링 계획 및 실행

**사용 시기:**
- 파일 구조 재구성
- 큰 컴포넌트 분리
- 파일 이동 후 import 경로 업데이트
- 코드 유지보수성 개선

**통합:** ✅ 그대로 복사

---

### documentation-architect
**목적:** 종합적인 문서 작성

**사용 시기:**
- 새 기능 문서화
- API 문서 작성
- 개발자 가이드 작성
- 아키텍처 개요 생성

**통합:** ✅ 그대로 복사

---

### frontend-error-fixer
**목적:** 프론트엔드 에러 디버깅 및 수정

**사용 시기:**
- 브라우저 콘솔 에러
- 프론트엔드 TypeScript 컴파일 에러
- React 에러
- 빌드 실패

**통합:** ⚠️ 스크린샷 경로 참조 가능 - 필요시 업데이트

---

### plan-reviewer
**목적:** 구현 전 개발 계획 리뷰

**사용 시기:**
- 복잡한 기능 시작 전
- 아키텍처 계획 검증
- 잠재적 문제 조기 식별
- 접근 방식에 대한 세컨드 오피니언 필요 시

**통합:** ✅ 그대로 복사

---

### refactor-planner
**목적:** 종합적인 리팩토링 전략 수립

**사용 시기:**
- 코드 재구성 계획
- 레거시 코드 현대화
- 큰 파일 분리
- 코드 구조 개선

**통합:** ✅ 그대로 복사

---

### web-research-specialist
**목적:** 인터넷에서 기술 이슈 조사

**사용 시기:**
- 난해한 에러 디버깅
- 문제 해결책 찾기
- 모범 사례 조사
- 구현 접근 방식 비교

**통합:** ✅ 그대로 복사

---

### auth-route-tester
**목적:** 인증된 API 엔드포인트 테스트

**사용 시기:**
- JWT 쿠키 인증을 사용하는 라우트 테스트
- 엔드포인트 기능 검증
- 인증 문제 디버깅

**통합:** ⚠️ JWT 쿠키 기반 인증 필요

---

### auth-route-debugger
**목적:** 인증 문제 디버깅

**사용 시기:**
- 인증 실패
- 토큰 문제
- 쿠키 문제
- 권한 에러

**통합:** ⚠️ JWT 쿠키 기반 인증 필요

---

### auto-error-resolver
**목적:** TypeScript 컴파일 에러 자동 수정

**사용 시기:**
- TypeScript 에러로 인한 빌드 실패
- 타입을 깨뜨리는 리팩토링 후
- 체계적인 에러 해결 필요 시

**통합:** ⚠️ 경로 업데이트 필요할 수 있음

---

## Agent 통합 방법

### 표준 통합 (대부분의 Agents)

**Step 1: 파일 복사**
```bash
cp showcase/.claude/agents/agent-name.md \\
   your-project/.claude/agents/
```

**Step 2: 확인 (선택사항)**
```bash
# 하드코딩된 경로 확인
grep -n "~/git/\|/root/git/\|/Users/" your-project/.claude/agents/agent-name.md
```

**Step 3: 사용하기**
Claude에게 요청: "[agent-name] agent를 사용해서 [작업]해줘"

끝! Agents는 즉시 작동합니다.

---

### 커스터마이징이 필요한 Agents

**frontend-error-fixer:**
- 스크린샷 경로 참조 가능
- 사용자에게 문의: "스크린샷을 어디에 저장할까요?"
- 에이전트 파일의 경로 업데이트

**auth-route-tester / auth-route-debugger:**
- JWT 쿠키 인증 필요
- 예제의 서비스 URL 업데이트
- 사용자의 인증 설정에 맞게 커스터마이징

**auto-error-resolver:**
- 하드코딩된 프로젝트 경로가 있을 수 있음
- `$CLAUDE_PROJECT_DIR` 또는 상대 경로 사용으로 업데이트

---

## Agents vs Skills 사용 시기

| Agents 사용 시... | Skills 사용 시... |
|-------------------|-------------------|
| 다단계 작업 필요 | 인라인 가이드 필요 |
| 복잡한 분석 필요 | 모범 사례 확인 |
| 자율적 작업 선호 | 제어권 유지 원함 |
| 명확한 최종 목표가 있는 작업 | 지속적인 개발 작업 |
| 예: "모든 컨트롤러 리뷰해줘" | 예: "새 라우트 만드는 중" |

**둘 다 함께 사용 가능:**
- Skill은 개발 중 패턴 제공
- Agent는 완료 후 결과 리뷰

---

## Agent 빠른 참조

| Agent | 복잡도 | 커스터마이징 | 인증 필요 |
|-------|--------|--------------|-----------|
| code-architecture-reviewer | 중간 | ✅ 없음 | 아니오 |
| code-refactor-master | 높음 | ✅ 없음 | 아니오 |
| documentation-architect | 중간 | ✅ 없음 | 아니오 |
| frontend-error-fixer | 중간 | ⚠️ 스크린샷 경로 | 아니오 |
| plan-reviewer | 낮음 | ✅ 없음 | 아니오 |
| refactor-planner | 중간 | ✅ 없음 | 아니오 |
| web-research-specialist | 낮음 | ✅ 없음 | 아니오 |
| auth-route-tester | 중간 | ⚠️ 인증 설정 | JWT 쿠키 |
| auth-route-debugger | 중간 | ⚠️ 인증 설정 | JWT 쿠키 |
| auto-error-resolver | 낮음 | ⚠️ 경로 | 아니오 |

---

## Claude Code를 위한 안내

**사용자를 위해 agents 통합 시:**

1. **[CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md) 읽기**
2. **.md 파일만 복사** - agents는 독립적
3. **하드코딩된 경로 확인:**
   ```bash
   grep "~/git/\|/root/" agent-name.md
   ```
4. **경로 발견 시 업데이트** - `$CLAUDE_PROJECT_DIR` 또는 `.`로
5. **인증 agents의 경우:** JWT 쿠키 인증 사용 여부 먼저 확인

**끝!** Agents는 가장 쉽게 통합할 수 있는 컴포넌트입니다.

---

## 나만의 Agents 만들기

Agents는 선택적 YAML frontmatter가 있는 마크다운 파일입니다:

```markdown
# Agent Name

## 목적
이 에이전트가 하는 일

## 지시사항
자율 실행을 위한 단계별 지시사항

## 사용 가능한 도구
이 에이전트가 사용할 수 있는 도구 목록

## 예상 출력
결과를 반환할 형식
```

**팁:**
- 지시사항을 매우 구체적으로 작성
- 복잡한 작업을 번호가 매겨진 단계로 분리
- 반환할 내용을 정확히 명시
- 좋은 출력 예제 포함
- 사용 가능한 도구 명시적으로 나열

---

## 문제 해결

### Agent를 찾을 수 없음

**확인:**
```bash
# 에이전트 파일이 있는지?
ls -la .claude/agents/[agent-name].md
```

### Agent가 경로 에러로 실패

**하드코딩된 경로 확인:**
```bash
grep "~/\|/root/\|/Users/" .claude/agents/[agent-name].md
```

**수정:**
```bash
sed -i 's|~/git/.*project|$CLAUDE_PROJECT_DIR|g' .claude/agents/[agent-name].md
```

---

## 다음 단계

1. **위의 agents 둘러보기** - 작업에 유용한 것 찾기
2. **필요한 것 복사** - .md 파일만
3. **Claude에게 사용 요청** - "[agent]를 사용해서 [작업]해줘"
4. **나만의 것 만들기** - 특정 요구사항에 맞게 패턴 따르기

**질문이 있으신가요?** [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md) 참조
