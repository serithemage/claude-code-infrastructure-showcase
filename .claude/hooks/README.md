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

## 필수 Hooks (여기서 시작)

### skill-activation-prompt (UserPromptSubmit)

**목적:** 사용자 프롬프트와 파일 컨텍스트를 기반으로 관련 skills를 자동으로 제안

**작동 방식:**
1. `skill-rules.json`을 읽음
2. 사용자 프롬프트를 트리거 패턴과 매칭
3. 사용자가 작업 중인 파일 확인
4. Claude의 컨텍스트에 skill 제안을 주입

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

**작동 방식:**
1. Edit/Write/MultiEdit 도구 호출 모니터링
2. 수정된 파일 기록
3. 컨텍스트 관리를 위한 캐시 생성
4. 프로젝트 구조 자동 감지 (frontend, backend, packages 등)

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

### tsc-check (Stop)

**목적:** 사용자가 중지할 때 TypeScript 컴파일 검사

**⚠️ 경고:** 멀티 서비스 모노레포 구조용으로 설정됨

**통합 방법:**

**먼저 이 hook이 적합한지 확인:**
- ✅ 사용 적합: 멀티 서비스 TypeScript 모노레포
- ❌ 건너뛰기: 단일 서비스 프로젝트 또는 다른 빌드 설정

**사용하는 경우:**
1. tsc-check.sh 복사
2. **서비스 감지 부분 수정 (약 28번째 줄):**
   ```bash
   # 예시 서비스를 실제 서비스로 교체:
   case "$repo" in
       api|web|auth|payments|...)  # ← 실제 서비스명
   ```
3. settings.json에 추가하기 전에 수동으로 테스트

**커스터마이징:** ⚠️⚠️⚠️ 많이 필요

---

### trigger-build-resolver (Stop)

**목적:** 컴파일 실패 시 build-error-resolver agent를 자동 실행

**의존성:** tsc-check hook이 정상 작동해야 함

**커스터마이징:** ✅ 불필요 (단, tsc-check가 먼저 작동해야 함)

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
