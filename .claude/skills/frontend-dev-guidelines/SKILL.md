---
name: frontend-dev-guidelines
description: React/TypeScript 애플리케이션을 위한 프론트엔드 개발 가이드라인. Suspense, lazy loading, useSuspenseQuery, features 디렉토리를 사용한 파일 구성, MUI v7 styling, TanStack Router, 성능 최적화, TypeScript 모범 사례를 포함한 최신 패턴. 컴포넌트, 페이지, 기능 생성, 데이터 fetching, styling, routing 또는 프론트엔드 코드 작업 시 사용.
---

# 프론트엔드 개발 가이드라인

## 목적

Suspense 기반 데이터 fetching, lazy loading, 적절한 파일 구성, 성능 최적화를 강조하는 최신 React 개발을 위한 종합 가이드입니다.

## 이 Skill 사용 시점

- 새 컴포넌트 또는 페이지 생성
- 새 기능 구축
- TanStack Query로 데이터 fetching
- TanStack Router로 routing 설정
- MUI v7로 컴포넌트 styling
- 성능 최적화
- 프론트엔드 코드 구성
- TypeScript 모범 사례

---

## 빠른 시작

### 새 컴포넌트 체크리스트

컴포넌트를 만드시나요? 이 체크리스트를 따르세요:

- [ ] TypeScript와 함께 `React.FC<Props>` 패턴 사용
- [ ] 무거운 컴포넌트인 경우 Lazy load: `React.lazy(() => import())`
- [ ] Loading 상태를 위해 `<SuspenseLoader>`로 래핑
- [ ] 데이터 fetching에 `useSuspenseQuery` 사용
- [ ] Import aliases: `@/`, `~types`, `~components`, `~features`
- [ ] 스타일: 100줄 미만이면 인라인, 100줄 초과면 별도 파일
- [ ] 자식에게 전달되는 이벤트 핸들러에 `useCallback` 사용
- [ ] 하단에 default export
- [ ] Loading 스피너를 사용한 early return 금지
- [ ] 사용자 알림에 `useMuiSnackbar` 사용

### 새 기능 체크리스트

기능을 만드시나요? 이 구조를 설정하세요:

- [ ] `features/{feature-name}/` 디렉토리 생성
- [ ] 하위 디렉토리 생성: `api/`, `components/`, `hooks/`, `helpers/`, `types/`
- [ ] API service 파일 생성: `api/{feature}Api.ts`
- [ ] `types/`에 TypeScript 타입 설정
- [ ] `routes/{feature-name}/index.tsx`에 route 생성
- [ ] 기능 컴포넌트 Lazy load
- [ ] Suspense boundaries 사용
- [ ] 기능 `index.ts`에서 public API export

---

## Import Aliases 빠른 참조

| Alias | 해석 | 예시 |
|-------|-------------|---------|
| `@/` | `src/` | `import { apiClient } from '@/lib/apiClient'` |
| `~types` | `src/types` | `import type { User } from '~types/user'` |
| `~components` | `src/components` | `import { SuspenseLoader } from '~components/SuspenseLoader'` |
| `~features` | `src/features` | `import { authApi } from '~features/auth'` |

정의 위치: [vite.config.ts](../../vite.config.ts) 180-185줄

---

## 공통 Imports 치트시트

```typescript
// React & Lazy Loading
import React, { useState, useCallback, useMemo } from 'react';
const Heavy = React.lazy(() => import('./Heavy'));

// MUI Components
import { Box, Paper, Typography, Button, Grid } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';

// TanStack Query (Suspense)
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';

// TanStack Router
import { createFileRoute } from '@tanstack/react-router';

// Project Components
import { SuspenseLoader } from '~components/SuspenseLoader';

// Hooks
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// Types
import type { Post } from '~types/post';
```

---

## 주제 가이드

### 🎨 컴포넌트 패턴

**최신 React 컴포넌트 사용:**
- 타입 안전성을 위한 `React.FC<Props>`
- 코드 분할을 위한 `React.lazy()`
- Loading 상태를 위한 `SuspenseLoader`
- Named const + default export 패턴

**핵심 개념:**
- 무거운 컴포넌트 Lazy load (DataGrid, 차트, 에디터)
- Lazy 컴포넌트는 항상 Suspense로 래핑
- SuspenseLoader 컴포넌트 사용 (fade 애니메이션 포함)
- 컴포넌트 구조: Props → Hooks → Handlers → Render → Export

**[📖 전체 가이드: resources/component-patterns.md](resources/component-patterns.md)**

---

### 📊 데이터 Fetching

**기본 패턴: useSuspenseQuery**
- Suspense boundaries와 함께 사용
- Cache-first 전략 (API 전에 grid 캐시 확인)
- `isLoading` 체크 대체
- 제네릭으로 타입 안전

**API Service 레이어:**
- `features/{feature}/api/{feature}Api.ts` 생성
- `apiClient` axios 인스턴스 사용
- 기능별 중앙화된 메서드
- Route 형식: `/form/route` (`/api/form/route` 아님)

**[📖 전체 가이드: resources/data-fetching.md](resources/data-fetching.md)**

---

### 📁 파일 구성

**features/ vs components/:**
- `features/`: 도메인 특화 (posts, comments, auth)
- `components/`: 진정으로 재사용 가능한 것 (SuspenseLoader, CustomAppBar)

**Feature 하위 디렉토리:**
```
features/
  my-feature/
    api/          # API service 레이어
    components/   # 기능 컴포넌트
    hooks/        # Custom hooks
    helpers/      # 유틸리티 함수
    types/        # TypeScript 타입
```

**[📖 전체 가이드: resources/file-organization.md](resources/file-organization.md)**

---

### 🎨 Styling

**인라인 vs 분리:**
- 100줄 미만: 인라인 `const styles: Record<string, SxProps<Theme>>`
- 100줄 초과: 별도 `.styles.ts` 파일

**기본 방법:**
- MUI 컴포넌트에 `sx` prop 사용
- `SxProps<Theme>`로 타입 안전
- Theme 접근: `(theme) => theme.palette.primary.main`

**MUI v7 Grid:**
```typescript
<Grid size={{ xs: 12, md: 6 }}>  // ✅ v7 문법
<Grid xs={12} md={6}>             // ❌ 이전 문법
```

**[📖 전체 가이드: resources/styling-guide.md](resources/styling-guide.md)**

---

### 🛣️ Routing

**TanStack Router - 폴더 기반:**
- 디렉토리: `routes/my-route/index.tsx`
- 컴포넌트 Lazy load
- `createFileRoute` 사용
- Loader에 Breadcrumb 데이터

**예시:**
```typescript
import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

export const Route = createFileRoute('/my-route/')({
    component: MyPage,
    loader: () => ({ crumb: 'My Route' }),
});
```

**[📖 전체 가이드: resources/routing-guide.md](resources/routing-guide.md)**

---

### ⏳ Loading & Error 상태

**핵심 규칙: Early Return 금지**

```typescript
// ❌ 절대 안 됨 - 레이아웃 시프트 유발
if (isLoading) {
    return <LoadingSpinner />;
}

// ✅ 항상 - 일관된 레이아웃
<SuspenseLoader>
    <Content />
</SuspenseLoader>
```

**이유:** Cumulative Layout Shift (CLS) 방지, 더 나은 UX

**Error Handling:**
- 사용자 피드백에 `useMuiSnackbar` 사용
- `react-toastify` 절대 사용 금지
- TanStack Query `onError` 콜백

**[📖 전체 가이드: resources/loading-and-error-states.md](resources/loading-and-error-states.md)**

---

### ⚡ 성능

**최적화 패턴:**
- `useMemo`: 비용이 큰 계산 (filter, sort, map)
- `useCallback`: 자식에게 전달되는 이벤트 핸들러
- `React.memo`: 비용이 큰 컴포넌트
- Debounced 검색 (300-500ms)
- 메모리 누수 방지 (useEffect에서 cleanup)

**[📖 전체 가이드: resources/performance.md](resources/performance.md)**

---

### 📘 TypeScript

**표준:**
- Strict 모드, `any` 타입 금지
- 함수에 명시적 반환 타입
- Type imports: `import type { User } from '~types/user'`
- JSDoc이 포함된 컴포넌트 prop 인터페이스

**[📖 전체 가이드: resources/typescript-standards.md](resources/typescript-standards.md)**

---

### 🔧 공통 패턴

**다루는 주제:**
- Zod 검증과 React Hook Form
- DataGrid wrapper 계약
- Dialog 컴포넌트 표준
- 현재 사용자를 위한 `useAuth` hook
- 캐시 무효화를 포함한 Mutation 패턴

**[📖 전체 가이드: resources/common-patterns.md](resources/common-patterns.md)**

---

### 📚 전체 예시

**작동하는 전체 예시:**
- 모든 패턴이 포함된 최신 컴포넌트
- 완전한 기능 구조
- API service 레이어
- Lazy loading이 포함된 Route
- Suspense + useSuspenseQuery
- 검증이 포함된 Form

**[📖 전체 가이드: resources/complete-examples.md](resources/complete-examples.md)**

---

## 네비게이션 가이드

| 필요한 작업... | 읽어야 할 리소스 |
|------------|-------------------|
| 컴포넌트 생성 | [component-patterns.md](resources/component-patterns.md) |
| 데이터 fetch | [data-fetching.md](resources/data-fetching.md) |
| 파일/폴더 구성 | [file-organization.md](resources/file-organization.md) |
| 컴포넌트 스타일링 | [styling-guide.md](resources/styling-guide.md) |
| Routing 설정 | [routing-guide.md](resources/routing-guide.md) |
| Loading/errors 처리 | [loading-and-error-states.md](resources/loading-and-error-states.md) |
| 성능 최적화 | [performance.md](resources/performance.md) |
| TypeScript 타입 | [typescript-standards.md](resources/typescript-standards.md) |
| Forms/Auth/DataGrid | [common-patterns.md](resources/common-patterns.md) |
| 전체 예시 보기 | [complete-examples.md](resources/complete-examples.md) |

---

## 핵심 원칙

1. **무거운 것은 모두 Lazy Load**: Routes, DataGrid, 차트, 에디터
2. **Loading에 Suspense**: early return 대신 SuspenseLoader 사용
3. **useSuspenseQuery**: 새 코드의 기본 데이터 fetching 패턴
4. **기능은 정리됨**: api/, components/, hooks/, helpers/ 하위 디렉토리
5. **크기에 따른 스타일**: 100줄 미만 인라인, 100줄 초과 분리
6. **Import Aliases**: @/, ~types, ~components, ~features 사용
7. **Early Return 금지**: 레이아웃 시프트 방지
8. **useMuiSnackbar**: 모든 사용자 알림에 사용

---

## 빠른 참조: 파일 구조

```
src/
  features/
    my-feature/
      api/
        myFeatureApi.ts       # API service
      components/
        MyFeature.tsx         # 메인 컴포넌트
        SubComponent.tsx      # 관련 컴포넌트
      hooks/
        useMyFeature.ts       # Custom hooks
        useSuspenseMyFeature.ts  # Suspense hooks
      helpers/
        myFeatureHelpers.ts   # 유틸리티
      types/
        index.ts              # TypeScript 타입
      index.ts                # Public exports

  components/
    SuspenseLoader/
      SuspenseLoader.tsx      # 재사용 가능한 loader
    CustomAppBar/
      CustomAppBar.tsx        # 재사용 가능한 app bar

  routes/
    my-route/
      index.tsx               # Route 컴포넌트
      create/
        index.tsx             # 중첩 route
```

---

## 최신 컴포넌트 템플릿 (빠른 복사)

```typescript
import React, { useState, useCallback } from 'react';
import { Box, Paper } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';
import { featureApi } from '../api/featureApi';
import type { FeatureData } from '~types/feature';

interface MyComponentProps {
    id: number;
    onAction?: () => void;
}

export const MyComponent: React.FC<MyComponentProps> = ({ id, onAction }) => {
    const [state, setState] = useState<string>('');

    const { data } = useSuspenseQuery({
        queryKey: ['feature', id],
        queryFn: () => featureApi.getFeature(id),
    });

    const handleAction = useCallback(() => {
        setState('updated');
        onAction?.();
    }, [onAction]);

    return (
        <Box sx={{ p: 2 }}>
            <Paper sx={{ p: 3 }}>
                {/* Content */}
            </Paper>
        </Box>
    );
};

export default MyComponent;
```

전체 예시는 [resources/complete-examples.md](resources/complete-examples.md)를 참조하세요

---

## 관련 Skills

- **error-tracking**: Sentry를 사용한 error tracking (프론트엔드에도 적용)
- **backend-dev-guidelines**: 프론트엔드가 소비하는 백엔드 API 패턴

---

**Skill 상태**: 최적의 context 관리를 위한 progressive loading이 포함된 모듈형 구조