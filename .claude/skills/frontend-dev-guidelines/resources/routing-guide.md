# Routing ガイド

TanStack Router 実装とフォルダベースルーティングおよび lazy loading パターンです。

---

## TanStack Router 概要

ファイルベースルーティングがある **TanStack Router**:
- フォルダ構造が routes を定義
- コード分割のための Lazy loading
- 型安全ルーティング
- Breadcrumb loaders

---

## フォルダベースルーティング

### ディレクトリ構造

```
routes/
  __root.tsx                    # ルートレイアウト
  index.tsx                     # ホーム route (/)
  posts/
    index.tsx                   # /posts
    create/
      index.tsx                 # /posts/create
    $postId.tsx                 # /posts/:postId (動的)
  comments/
    index.tsx                   # /comments
```

**パターン**:
- `index.tsx` = そのパスの Route
- `$param.tsx` = 動的パラメータ
- ネストフォルダ = ネスト routes

---

## 基本 Route パターン

### posts/index.tsx 例

```typescript
/**
 * Posts route コンポーネント
 * メインブログポストリスト表示
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

// ページコンポーネント Lazy load
const PostsList = lazy(() =>
    import('@/features/posts/components/PostsList').then(
        (module) => ({ default: module.PostsList }),
    ),
);

export const Route = createFileRoute('/posts/')(({
    component: PostsPage,
    // Breadcrumb データ定義
    loader: () => ({
        crumb: 'Posts',
    }),
});

function PostsPage() {
    return (
        <PostsList
            title='All Posts'
            showFilters={true}
        />
    );
}

export default PostsPage;
```

**キーポイント:**
- 重いコンポーネント lazy load
- route パスと共に `createFileRoute`
- breadcrumb データ用 `loader`
- Page コンポーネントがコンテンツレンダー
- Route とコンポーネント両方を export

---

## Lazy Loading Routes

### Named Export パターン

```typescript
import { lazy } from 'react';

// named exports の場合 .then() で default にマッピング
const MyPage = lazy(() =>
    import('@/features/my-feature/components/MyPage').then(
        (module) => ({ default: module.MyPage })
    )
);
```

### Default Export パターン

```typescript
import { lazy } from 'react';

// default exports の場合より簡単な文法
const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));
```

### Routes を Lazy Load する理由

- コード分割 - より小さい初期バンドル
- より速い初期ページロード
- ナビゲーション時のみ route コードロード
- より良いパフォーマンス

---

## createFileRoute

### 基本設定

```typescript
export const Route = createFileRoute('/my-route/')(({
    component: MyRoutePage,
});

function MyRoutePage() {
    return <div>My Route Content</div>;
}
```

### Breadcrumb Loader と共に

```typescript
export const Route = createFileRoute('/my-route/')(({
    component: MyRoutePage,
    loader: () => ({
        crumb: 'My Route Title',
    }),
});
```

Breadcrumb がナビゲーション/アプリバーに自動的に表示されます。

### データ Loader と共に

```typescript
export const Route = createFileRoute('/my-route/')(({
    component: MyRoutePage,
    loader: async () => {
        // ここでデータ prefetch 可能
        const data = await api.getData();
        return { crumb: 'My Route', data };
    },
});
```

### Search Params と共に

```typescript
export const Route = createFileRoute('/search/')(({
    component: SearchPage,
    validateSearch: (search: Record<string, unknown>) => {
        return {
            query: (search.query as string) || '',
            page: Number(search.page) || 1,
        };
    },
});

function SearchPage() {
    const { query, page } = Route.useSearch();
    // query と page 使用
}
```

---

## 動的 Routes

### パラメータ Routes

```typescript
// routes/users/$userId.tsx

export const Route = createFileRoute('/users/$userId')(({
    component: UserPage,
});

function UserPage() {
    const { userId } = Route.useParams();

    return <UserProfile userId={userId} />;
}
```

### 複数パラメータ

```typescript
// routes/posts/$postId/comments/$commentId.tsx

export const Route = createFileRoute('/posts/$postId/comments/$commentId')(({
    component: CommentPage,
});

function CommentPage() {
    const { postId, commentId } = Route.useParams();

    return <CommentEditor postId={postId} commentId={commentId} />;
}
```

---

## ナビゲーション

### プログラマティックナビゲーション

```typescript
import { useNavigate } from '@tanstack/react-router';

export const MyComponent: React.FC = () => {
    const navigate = useNavigate();

    const handleClick = () => {
        navigate({ to: '/posts' });
    };

    return <Button onClick={handleClick}>View Posts</Button>;
};
```

### パラメータと共に

```typescript
const handleNavigate = () => {
    navigate({
        to: '/users/$userId',
        params: { userId: '123' },
    });
};
```

### Search Params と共に

```typescript
const handleSearch = () => {
    navigate({
        to: '/search',
        search: { query: 'test', page: 1 },
    });
};
```

---

## Route レイアウトパターン

### ルートレイアウト (__root.tsx)

```typescript
import { createRootRoute, Outlet } from '@tanstack/react-router';
import { Box } from '@mui/material';
import { CustomAppBar } from '~components/CustomAppBar';

export const Route = createRootRoute({
    component: RootLayout,
});

function RootLayout() {
    return (
        <Box>
            <CustomAppBar />
            <Box sx={{ p: 2 }}>
                <Outlet />  {/* 子 routes がここでレンダー */}
            </Box>
        </Box>
    );
}
```

### ネストレイアウト

```typescript
// routes/dashboard/index.tsx
export const Route = createFileRoute('/dashboard/')(({
    component: DashboardLayout,
});

function DashboardLayout() {
    return (
        <Box>
            <DashboardSidebar />
            <Box sx={{ flex: 1 }}>
                <Outlet />  {/* ネスト routes */}
            </Box>
        </Box>
    );
}
```

---

## 完全な Route 例

```typescript
/**
 * User profile route
 * パス: /users/:userId
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';
import { SuspenseLoader } from '~components/SuspenseLoader';

// 重いコンポーネント Lazy load
const UserProfile = lazy(() =>
    import('@/features/users/components/UserProfile').then(
        (module) => ({ default: module.UserProfile })
    )
);

export const Route = createFileRoute('/users/$userId')(({
    component: UserPage,
    loader: () => ({
        crumb: 'User Profile',
    }),
});

function UserPage() {
    const { userId } = Route.useParams();

    return (
        <SuspenseLoader>
            <UserProfile userId={userId} />
        </SuspenseLoader>
    );
}

export default UserPage;
```

---

## まとめ

**Routing チェックリスト:**
- ✅ フォルダベース: `routes/my-route/index.tsx`
- ✅ コンポーネント Lazy load: `React.lazy(() => import())`
- ✅ route パスと共に `createFileRoute` 使用
- ✅ `loader` 関数に breadcrumb 追加
- ✅ ローディング状態のために `SuspenseLoader` でラップ
- ✅ 動的 params に `Route.useParams()` 使用
- ✅ プログラマティックナビゲーションに `useNavigate()` 使用

**参考:**
- [component-patterns.md](component-patterns.md) - Lazy loading パターン
- [loading-and-error-states.md](loading-and-error-states.md) - SuspenseLoader 使用
- [complete-examples.md](complete-examples.md) - 完全 route 例
