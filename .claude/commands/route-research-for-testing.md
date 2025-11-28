---
description: 수정된 라우트 매핑 및 테스트 실행
argument-hint: "[/extra/path …]"
allowed-tools: Bash(cat:*), Bash(awk:*), Bash(grep:*), Bash(sort:*), Bash(xargs:*), Bash(sed:*)
model: sonnet
---

## 컨텍스트

이번 세션에서 변경된 라우트 파일 (자동 생성):

!cat "$CLAUDE_PROJECT_DIR/.claude/tsc-cache"/\*/edited-files.log \
 | awk -F: '{print $2}' \
 | grep '/routes/' \
 | sort -u

사용자 지정 추가 라우트: `$ARGUMENTS`

## 수행할 작업

다음 단계를 **정확히** 따르세요:

1. 자동 목록과 `$ARGUMENTS`를 결합하고, 중복을 제거하고, `src/app.ts`에
   정의된 접두사를 해결합니다.
2. 각 최종 라우트에 대해 경로, 메서드, 예상 요청/응답 형태,
   유효한 페이로드와 유효하지 않은 페이로드 예시가 포함된 JSON 레코드를 출력합니다.
3. **이제 `Task` 도구를 호출**하세요:

```json
{
    "tool": "Task",
    "parameters": {
        "description": "route smoke tests",
        "prompt": "Run the auth-route-tester sub-agent on the JSON above."
    }
}
```
