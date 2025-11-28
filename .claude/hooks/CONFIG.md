# Hooks 설정 가이드

이 가이드는 프로젝트에서 hooks 시스템을 설정하고 커스터마이징하는 방법을 설명합니다.

## 빠른 시작 설정

### 1. .claude/settings.json에 Hooks 등록

프로젝트 루트에 `.claude/settings.json`을 생성하거나 업데이트하세요:

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
    ],
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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/error-handling-reminder.sh"
          }
        ]
      }
    ]
  }
}
```

### 2. 의존성 설치

```bash
cd .claude/hooks
npm install
```

### 3. 실행 권한 설정

```bash
chmod +x .claude/hooks/*.sh
```

## 커스터마이징 옵션

### 프로젝트 구조 감지

기본적으로 hooks는 다음 디렉토리 패턴을 감지합니다:

**Frontend:** `frontend/`, `client/`, `web/`, `app/`, `ui/`
**Backend:** `backend/`, `server/`, `api/`, `src/`, `services/`
**Database:** `database/`, `prisma/`, `migrations/`
**Monorepo:** `packages/*`, `examples/*`

#### 커스텀 디렉토리 패턴 추가

`.claude/hooks/post-tool-use-tracker.sh`의 `detect_repo()` 함수를 수정하세요:

```bash
case "$repo" in
    # 여기에 커스텀 디렉토리 추가
    my-custom-service)
        echo "$repo"
        ;;
    admin-panel)
        echo "$repo"
        ;;
    # ... 기존 패턴
esac
```

### 빌드 명령어 감지

Hooks는 다음을 기반으로 빌드 명령어를 자동 감지합니다:
1. "build" 스크립트가 있는 `package.json`의 존재 여부
2. 패키지 매니저 (pnpm > npm > yarn)
3. 특수 케이스 (Prisma 스키마)

#### 빌드 명령어 커스터마이징

`.claude/hooks/post-tool-use-tracker.sh`의 `get_build_command()` 함수를 수정하세요:

```bash
# 커스텀 빌드 로직 추가
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && make build"
    return
fi
```

### TypeScript 설정

Hooks는 자동으로 감지합니다:
- 표준 TypeScript 프로젝트용 `tsconfig.json`
- Vite/React 프로젝트용 `tsconfig.app.json`

#### 커스텀 TypeScript 설정

`.claude/hooks/post-tool-use-tracker.sh`의 `get_tsc_command()` 함수를 수정하세요:

```bash
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && npx tsc --project tsconfig.build.json --noEmit"
    return
fi
```

### Prettier 설정

Prettier hook은 다음 순서로 설정을 검색합니다:
1. 현재 파일 디렉토리 (상위로 탐색)
2. 프로젝트 루트
3. Prettier 기본값으로 폴백

#### 커스텀 Prettier 설정 검색

`.claude/hooks/stop-prettier-formatter.sh`의 `get_prettier_config()` 함수를 수정하세요:

```bash
# 커스텀 설정 위치 추가
if [[ -f "$project_root/config/.prettierrc" ]]; then
    echo "$project_root/config/.prettierrc"
    return
fi
```

### 에러 처리 리마인더

`.claude/hooks/error-handling-reminder.ts`에서 파일 카테고리 감지를 설정하세요:

```typescript
function getFileCategory(filePath: string): 'backend' | 'frontend' | 'database' | 'other' {
    // 커스텀 패턴 추가
    if (filePath.includes('/my-custom-dir/')) return 'backend';
    // ... 기존 패턴
}
```

### 에러 임계값 설정

auto-error-resolver agent 추천 시점을 변경합니다.

`.claude/hooks/stop-build-check-enhanced.sh`를 수정하세요:

```bash
# 기본값은 5개 에러 - 원하는 값으로 변경
if [[ $total_errors -ge 10 ]]; then  # 이제 10개 이상의 에러 필요
    # agent 추천
fi
```

## 환경 변수

### 전역 환경 변수

쉘 프로필 (`.bashrc`, `.zshrc` 등)에 설정하세요:

```bash
# 에러 처리 리마인더 비활성화
export SKIP_ERROR_REMINDER=1

# 커스텀 프로젝트 디렉토리 (기본값을 사용하지 않는 경우)
export CLAUDE_PROJECT_DIR=/path/to/your/project
```

### 세션별 환경 변수

Claude Code 시작 전에 설정하세요:

```bash
SKIP_ERROR_REMINDER=1 claude-code
```

## Hook 실행 순서

Stop hooks는 `settings.json`에 지정된 순서대로 실행됩니다:

```json
"Stop": [
  {
    "hooks": [
      { "command": "...formatter.sh" },    // 첫 번째로 실행
      { "command": "...build-check.sh" },  // 두 번째로 실행
      { "command": "...reminder.sh" }      // 세 번째로 실행
    ]
  }
]
```

**이 순서가 중요한 이유:**
1. 먼저 파일 포맷팅 (깔끔한 코드)
2. 그 다음 에러 확인
3. 마지막으로 리마인더 표시

## 선택적 Hook 활성화

모든 hooks가 필요하지는 않습니다. 프로젝트에 맞는 것을 선택하세요:

### 최소 설정 (Skill 활성화만)

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

### 빌드 검사만 (포맷팅 없음)

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
    ],
    "Stop": [
      {
        "hooks": [
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

### 포맷팅만 (빌드 검사 없음)

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          }
        ]
      }
    ]
  }
}
```

## 캐시 관리

### 캐시 위치

```
$CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session_id]/
```

### 수동 캐시 정리

```bash
# 모든 캐시 데이터 제거
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/*

# 특정 세션 제거
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session-id]
```

### 자동 정리

build-check hook은 빌드 성공 시 세션 캐시를 자동으로 정리합니다.

## 설정 문제 해결

### Hook이 실행되지 않음

1. **등록 확인:** `.claude/settings.json`에 hook이 있는지 확인
2. **권한 확인:** `chmod +x .claude/hooks/*.sh` 실행
3. **경로 확인:** `$CLAUDE_PROJECT_DIR`이 올바르게 설정되었는지 확인
4. **TypeScript 확인:** `cd .claude/hooks && npx tsc`를 실행하여 에러 확인

### 잘못된 감지 (False Positive)

**문제:** Hook이 처리하지 않아야 할 파일에서 트리거됨

**해결:** 해당 hook에 스킵 조건 추가:

```bash
# post-tool-use-tracker.sh 내에서
if [[ "$file_path" =~ /generated/ ]]; then
    exit 0  # 생성된 파일 스킵
fi
```

### 성능 문제

**문제:** Hooks가 느림

**해결책:**
1. TypeScript 검사를 변경된 파일로만 제한
2. 더 빠른 패키지 매니저 사용 (pnpm > npm)
3. 스킵 조건 추가
4. 큰 파일에서 Prettier 비활성화

```bash
# stop-prettier-formatter.sh에서 큰 파일 스킵
file_size=$(wc -c < "$file" 2>/dev/null || echo 0)
if [[ $file_size -gt 100000 ]]; then  # 100KB보다 큰 파일 스킵
    continue
fi
```

### Hooks 디버깅

어떤 hook에든 디버그 출력 추가:

```bash
# hook 스크립트 상단에
set -x  # 디버그 모드 활성화

# 또는 특정 디버그 라인 추가
echo "DEBUG: file_path=$file_path" >&2
echo "DEBUG: repo=$repo" >&2
```

Claude Code 로그에서 hook 실행을 확인하세요.

## 고급 설정

### 커스텀 Hook 이벤트 핸들러

다른 이벤트용으로 자체 hooks를 만들 수 있습니다:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/my-custom-bash-guard.sh"
          }
        ]
      }
    ]
  }
}
```

### 모노레포 설정

여러 패키지가 있는 모노레포의 경우:

```bash
# post-tool-use-tracker.sh의 detect_repo() 내에서
case "$repo" in
    packages)
        # 패키지명 가져오기
        local package=$(echo "$relative_path" | cut -d'/' -f2)
        if [[ -n "$package" ]]; then
            echo "packages/$package"
        else
            echo "$repo"
        fi
        ;;
esac
```

### Docker/컨테이너 프로젝트

빌드 명령어가 컨테이너에서 실행되어야 하는 경우:

```bash
# post-tool-use-tracker.sh의 get_build_command() 내에서
if [[ "$repo" == "api" ]]; then
    echo "docker-compose exec api npm run build"
    return
fi
```

## 모범 사례

1. **최소한으로 시작** - hooks를 하나씩 활성화
2. **철저히 테스트** - 변경 후 hooks 작동 확인
3. **커스터마이징 문서화** - 커스텀 로직을 설명하는 주석 추가
4. **버전 관리** - `.claude/` 디렉토리를 git에 커밋
5. **팀 일관성** - 팀 전체에 설정 공유

## 참고 자료

- [README.md](./README.md) - Hooks 개요
- [../../docs/HOOKS_SYSTEM.md](../../docs/HOOKS_SYSTEM.md) - 완전한 hooks 레퍼런스
- [../../docs/SKILLS_SYSTEM.md](../../docs/SKILLS_SYSTEM.md) - Skills 통합
