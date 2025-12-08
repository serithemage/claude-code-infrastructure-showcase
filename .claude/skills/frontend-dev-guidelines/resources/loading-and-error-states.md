# Loading & Error States

**重要**: 適切な loading と error state 処理はレイアウト移動を防止し、より良いユーザー体験を提供します。

---

## ⚠️ 核心ルール: Early Return 絶対禁止

### 問題

```typescript
// ❌ 絶対にこうしないでください - loading spinner で early return
const Component = () => {
    const { data, isLoading } = useQuery();

    // 間違い: レイアウト移動と悪い UX を引き起こす
    if (isLoading) {
        return <LoadingSpinner />;
    }

    return <Content data={data} />;
};
```

**なぜ悪いか:**
1. **レイアウト移動**: ローディング完了時にコンテンツ位置がジャンプ
2. **CLS (Cumulative Layout Shift)**: 悪い Core Web Vital スコア
3. **不快な UX**: ページ構造が突然変更
4. **スクロール位置損失**: ユーザーがページ内の位置を失う

### 解決策

**オプション 1: SuspenseLoader (新コンポーネントに推奨)**

```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';

const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

export const MyComponent: React.FC = () => {
    return (
        <SuspenseLoader>
            <HeavyComponent />
        </SuspenseLoader>
    );
};
```

**オプション 2: LoadingOverlay (レガシー useQuery パターン用)**

```typescript
import { LoadingOverlay } from '~components/LoadingOverlay';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({ ... });

    return (
        <LoadingOverlay loading={isLoading}>
            <Content data={data} />
        </LoadingOverlay>
    );
};
```

---

## SuspenseLoader コンポーネント

### 機能

- lazy コンポーネントローディング中にローディングインジケーター表示
- スムーズなフェードインアニメーション
- レイアウト移動防止
- アプリ全体で一貫したローディング体験

### Import

```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
// または
import { SuspenseLoader } from '@/components/SuspenseLoader';
```

### 基本使用法

```typescript
<SuspenseLoader>
    <LazyLoadedComponent />
</SuspenseLoader>
```

### useSuspenseQuery と共に

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { SuspenseLoader } from '~components/SuspenseLoader';

const Inner: React.FC = () => {
    // isLoading 不要!
    const { data } = useSuspenseQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),
    });

    return <Display data={data} />;
};

// 外部コンポーネントが Suspense でラップ
export const Outer: React.FC = () => {
    return (
        <SuspenseLoader>
            <Inner />
        </SuspenseLoader>
    );
};
```

### 複数 Suspense Boundary

**パターン**: 独立したセクションに対して別々のローディング

```typescript
export const Dashboard: React.FC = () => {
    return (
        <Box>
            <SuspenseLoader>
                <Header />
            </SuspenseLoader>

            <SuspenseLoader>
                <MainContent />
            </SuspenseLoader>

            <SuspenseLoader>
                <Sidebar />
            </SuspenseLoader>
        </Box>
    );
};
```

**利点:**
- 各セクションが独立してロード
- ユーザーが部分コンテンツをより早く見れる
- より良い体感パフォーマンス

### ネスト Suspense

```typescript
export const ParentComponent: React.FC = () => {
    return (
        <SuspenseLoader>
            {/* Parent がローディング中に suspend */}
            <ParentContent>
                <SuspenseLoader>
                    {/* child のためのネスト suspense */}
                    <ChildComponent />
                </SuspenseLoader>
            </ParentContent>
        </SuspenseLoader>
    );
};
```

---

## LoadingOverlay コンポーネント

### 使用タイミング

- `useQuery` があるレガシーコンポーネント (まだ Suspense にリファクタリングされていない)
- オーバーレイローディング状態が必要な時
- Suspense boundary が使用できない時

### 使用法

```typescript
import { LoadingOverlay } from '~components/LoadingOverlay';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),
    });

    return (
        <LoadingOverlay loading={isLoading}>
            <Box sx={{ p: 2 }}>
                {data && <Content data={data} />}
            </Box>
        </LoadingOverlay>
    );
};
```

**機能:**
- スピナーがある半透明オーバーレイ表示
- コンテンツ領域を予約 (レイアウト移動なし)
- ローディング中のインタラクション防止

---

## エラー処理

### useMuiSnackbar Hook (必須)

**絶対に react-toastify 使用禁止** - プロジェクト標準は MUI Snackbar

```typescript
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const { showSuccess, showError, showInfo, showWarning } = useMuiSnackbar();

    const handleAction = async () => {
        try {
            await api.doSomething();
            showSuccess('Operation completed successfully');
        } catch (error) {
            showError('Operation failed');
        }
    };

    return <Button onClick={handleAction}>Do Action</Button>;
};
```

**利用可能なメソッド:**
- `showSuccess(message)` - 緑の成功メッセージ
- `showError(message)` - 赤のエラーメッセージ
- `showWarning(message)` - オレンジの警告メッセージ
- `showInfo(message)` - 青の情報メッセージ

### TanStack Query Error Callbacks

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const { showError } = useMuiSnackbar();

    const { data } = useSuspenseQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),

        // エラー処理
        onError: (error) => {
            showError('Failed to load data');
            console.error('Query error:', error);
        },
    });

    return <Content data={data} />;
};
```

### Error Boundaries

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
    return (
        <Box sx={{ p: 4, textAlign: 'center' }}>
            <Typography variant='h5' color='error'>
                Something went wrong
            </Typography>
            <Typography>{error.message}</Typography>
            <Button onClick={resetErrorBoundary}>Try Again</Button>
        </Box>
    );
}

export const MyPage: React.FC = () => {
    return (
        <ErrorBoundary
            FallbackComponent={ErrorFallback}
            onError={(error) => console.error('Boundary caught:', error)}
        >
            <SuspenseLoader>
                <ComponentThatMightError />
            </SuspenseLoader>
        </ErrorBoundary>
    );
};
```

---

## 完全な例

### 例 1: Suspense がある最新コンポーネント

```typescript
import React from 'react';
import { Box, Paper } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { myFeatureApi } from '../api/myFeatureApi';

// Inner コンポーネントは useSuspenseQuery 使用
const InnerComponent: React.FC<{ id: number }> = ({ id }) => {
    const { data } = useSuspenseQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    // data は常に定義済み - isLoading 不要!
    return (
        <Paper sx={{ p: 2 }}>
            <h2>{data.title}</h2>
            <p>{data.description}</p>
        </Paper>
    );
};

// Outer コンポーネントが Suspense boundary 提供
export const OuterComponent: React.FC<{ id: number }> = ({ id }) => {
    return (
        <Box>
            <SuspenseLoader>
                <InnerComponent id={id} />
            </SuspenseLoader>
        </Box>
    );
};

export default OuterComponent;
```

### 例 2: LoadingOverlay があるレガシーパターン

```typescript
import React from 'react';
import { Box } from '@mui/material';
import { useQuery } from '@tanstack/react-query';
import { LoadingOverlay } from '~components/LoadingOverlay';
import { myFeatureApi } from '../api/myFeatureApi';

export const LegacyComponent: React.FC<{ id: number }> = ({ id }) => {
    const { data, isLoading, error } = useQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    return (
        <LoadingOverlay loading={isLoading}>
            <Box sx={{ p: 2 }}>
                {error && <ErrorDisplay error={error} />}
                {data && <Content data={data} />}
            </Box>
        </LoadingOverlay>
    );
};
```

### 例 3: Snackbar があるエラー処理

```typescript
import React from 'react';
import { useSuspenseQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Button } from '@mui/material';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import { myFeatureApi } from '../api/myFeatureApi';

export const EntityEditor: React.FC<{ id: number }> = ({ id }) => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const { data } = useSuspenseQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
        onError: () => {
            showError('Failed to load entity');
        },
    });

    const updateMutation = useMutation({
        mutationFn: (updates) => myFeatureApi.update(id, updates),

        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['entity', id] });
            showSuccess('Entity updated successfully');
        },

        onError: () => {
            showError('Failed to update entity');
        },
    });

    return (
        <Button onClick={() => updateMutation.mutate({ name: 'New' })}>
            Update
        </Button>
    );
};
```

---

## Loading State アンチパターン

### ❌ すべきでないこと

```typescript
// ❌ 絶対禁止 - Early return
if (isLoading) {
    return <CircularProgress />;
}

// ❌ 絶対禁止 - 条件付きレンダリング
{isLoading ? <Spinner /> : <Content />}

// ❌ 絶対禁止 - レイアウト変更
if (isLoading) {
    return (
        <Box sx={{ height: 100 }}>
            <Spinner />
        </Box>
    );
}
return (
    <Box sx={{ height: 500 }}>  // 異なる高さ!
        <Content />
    </Box>
);
```

### ✅ すべきこと

```typescript
// ✅ 最善 - useSuspenseQuery + SuspenseLoader
<SuspenseLoader>
    <ComponentWithSuspenseQuery />
</SuspenseLoader>

// ✅ 許容 - LoadingOverlay
<LoadingOverlay loading={isLoading}>
    <Content />
</LoadingOverlay>

// ✅ OK - 同じレイアウトのインライン skeleton
<Box sx={{ height: 500 }}>
    {isLoading ? <Skeleton variant='rectangular' height='100%' /> : <Content />}
</Box>
```

---

## Skeleton Loading (代替)

### MUI Skeleton コンポーネント

```typescript
import { Skeleton, Box } from '@mui/material';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({ ... });

    return (
        <Box sx={{ p: 2 }}>
            {isLoading ? (
                <>
                    <Skeleton variant='text' width={200} height={40} />
                    <Skeleton variant='rectangular' width='100%' height={200} />
                    <Skeleton variant='text' width='100%' />
                </>
            ) : (
                <>
                    <Typography variant='h5'>{data.title}</Typography>
                    <img src={data.image} />
                    <Typography>{data.description}</Typography>
                </>
            )}
        </Box>
    );
};
```

**核心**: Skeleton は実際のコンテンツと**同じレイアウト**でなければならない (移動なし)

---

## まとめ

**Loading States:**
- ✅ **推奨**: SuspenseLoader + useSuspenseQuery (最新パターン)
- ✅ **許容**: LoadingOverlay (レガシーパターン)
- ✅ **OK**: 同じレイアウトの Skeleton
- ❌ **絶対禁止**: Early return または条件付きレイアウト

**エラー処理:**
- ✅ **常に**: ユーザーフィードバックに useMuiSnackbar
- ❌ **絶対禁止**: react-toastify
- ✅ queries/mutations で onError コールバック使用
- ✅ コンポーネントレベルエラーに Error boundaries

**参考:**
- [component-patterns.md](component-patterns.md) - Suspense 統合
- [data-fetching.md](data-fetching.md) - useSuspenseQuery 詳細
