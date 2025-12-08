# データフェッチングパターン

Suspense boundaries、cache-first 戦略、中央化された API サービスを使用した TanStack Query ベースの最新データフェッチングパターンです。

---

## 主要パターン: useSuspenseQuery

### useSuspenseQuery を使用する理由

**すべての新コンポーネント**では一般的な `useQuery` の代わりに `useSuspenseQuery` を使用してください:

**利点:**
- `isLoading` チェックが不要
- Suspense boundaries と統合
- よりクリーンなコンポーネントコード
- 一貫したローディング UX
- Error boundaries を通じたより良いエラー処理

### 基本パターン

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { myFeatureApi } from '../api/myFeatureApi';

export const MyComponent: React.FC<Props> = ({ id }) => {
    // isLoading 不要 - Suspense が処理!
    const { data } = useSuspenseQuery({
        queryKey: ['myEntity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    // data は常に定義済み (undefined | Data ではない)
    return <div>{data.name}</div>;
};

// Suspense boundary でラップ
<SuspenseLoader>
    <MyComponent id={123} />
</SuspenseLoader>
```

### useSuspenseQuery vs useQuery

| 特徴 | useSuspenseQuery | useQuery |
|------|------------------|----------|
| ローディング状態 | Suspense が処理 | 手動 `isLoading` チェック |
| データ型 | 常に定義済み | `Data \| undefined` |
| 使用対象 | Suspense boundaries | 従来型コンポーネント |
| 推奨対象 | **新コンポーネント** | レガシーコードのみ |
| エラー処理 | Error boundaries | 手動エラー状態 |

**通常の useQuery を使用すべき時:**
- レガシーコードの保守
- Suspense なしの非常に単純なケース
- バックグラウンド更新のあるポーリング

**新コンポーネント: 常に useSuspenseQuery を優先**

---

## Cache-First 戦略

### Cache-First パターン例

**スマートキャッシング**で React Query キャッシュを先に確認して API 呼び出しを削減:

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';

export function useSuspensePost(postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery({
        queryKey: ['post', postId],
        queryFn: async () => {
            // 戦略 1: リストキャッシュから先に取得を試みる
            const cachedListData = queryClient.getQueryData<{ posts: Post[] }>([
                'posts',
                'list'
            ]);

            if (cachedListData?.posts) {
                const cachedPost = cachedListData.posts.find(
                    (post) => post.id === postId
                );

                if (cachedPost) {
                    return cachedPost;  // キャッシュから返却!
                }
            }

            // 戦略 2: キャッシュにない、API から fetch
            return postApi.getPost(postId);
        },
        staleTime: 5 * 60 * 1000,      // 5分間 fresh として扱う
        gcTime: 10 * 60 * 1000,         // 10分間キャッシュに保持
        refetchOnWindowFocus: false,    // フォーカス時に refetch しない
    });
}
```

**キーポイント:**
- API 呼び出し前にグリッド/リストキャッシュを確認
- 重複リクエストを防止
- `staleTime`: データが fresh として扱われる期間
- `gcTime`: 使用されないデータがキャッシュに残る期間
- `refetchOnWindowFocus: false`: ユーザー設定

---

## 並列データフェッチング

### useSuspenseQueries

複数の独立したリソースを fetch する場合:

```typescript
import { useSuspenseQueries } from '@tanstack/react-query';

export const MyComponent: React.FC = () => {
    const [userQuery, settingsQuery, preferencesQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['user'],
                queryFn: () => userApi.getCurrentUser(),
            },
            {
                queryKey: ['settings'],
                queryFn: () => settingsApi.getSettings(),
            },
            {
                queryKey: ['preferences'],
                queryFn: () => preferencesApi.getPreferences(),
            },
        ],
    });

    // すべてのデータ利用可能、Suspense がローディング処理
    const user = userQuery.data;
    const settings = settingsQuery.data;
    const preferences = preferencesQuery.data;

    return <Display user={user} settings={settings} prefs={preferences} />;
};
```

**利点:**
- すべてのクエリが並列実行
- 単一 Suspense boundary
- 型安全な結果

---

## Query Keys 構成

### ネーミング規則

```typescript
// Entity リスト
['entities', blogId]
['entities', blogId, 'summary']    // ビューモードと共に
['entities', blogId, 'flat']

// 単一 entity
['entity', blogId, entityId]

// 関連データ
['entity', entityId, 'history']
['entity', entityId, 'comments']

// ユーザー別
['user', userId, 'profile']
['user', userId, 'permissions']
```

**ルール:**
- entity 名で開始 (リストは複数形、単一は単数形)
- 特定性のために ID を含める
- ビューモード / 関係は末尾に追加
- アプリ全体で一貫性を維持

### Query Key 例

```typescript
// useSuspensePost.ts から
queryKey: ['post', blogId, postId]
queryKey: ['posts-v2', blogId, 'summary']

// Invalidation パターン
queryClient.invalidateQueries({ queryKey: ['post', blogId] });  // form のすべての posts
queryClient.invalidateQueries({ queryKey: ['post'] });          // すべての posts
```

---

## API サービスレイヤーパターン

### ファイル構造

feature ごとに中央化された API サービスを作成:

```
features/
  my-feature/
    api/
      myFeatureApi.ts    # サービスレイヤー
```

### サービスパターン (postApi.ts ベース)

```typescript
/**
 * my-feature 操作のための中央化された API サービス
 * 一貫したエラー処理のために apiClient を使用
 */
import apiClient from '@/lib/apiClient';
import type { MyEntity, UpdatePayload } from '../types';

export const myFeatureApi = {
    /**
     * 単一 entity fetch
     */
    getEntity: async (blogId: number, entityId: number): Promise<MyEntity> => {
        const { data } = await apiClient.get(
            `/blog/entities/${blogId}/${entityId}`
        );
        return data;
    },

    /**
     * form のすべての entities fetch
     */
    getEntities: async (blogId: number, view: 'summary' | 'flat'): Promise<MyEntity[]> => {
        const { data } = await apiClient.get(
            `/blog/entities/${blogId}`,
            { params: { view } }
        );
        return data.rows;
    },

    /**
     * Entity 更新
     */
    updateEntity: async (
        blogId: number,
        entityId: number,
        payload: UpdatePayload
    ): Promise<MyEntity> => {
        const { data } = await apiClient.put(
            `/blog/entities/${blogId}/${entityId}`,
            payload
        );
        return data;
    },

    /**
     * Entity 削除
     */
    deleteEntity: async (blogId: number, entityId: number): Promise<void> => {
        await apiClient.delete(`/blog/entities/${blogId}/${entityId}`);
    },
};
```

**キーポイント:**
- メソッドを持つ単一オブジェクト export
- `apiClient` 使用 (`@/lib/apiClient` の axios インスタンス)
- 型安全なパラメータと戻り値
- 各メソッドに JSDoc コメント
- 中央化されたエラー処理 (apiClient が処理)

---

## Route 形式規則 (重要)

### 正しい形式

```typescript
// ✅ 正しい - 直接サービスパス
await apiClient.get('/blog/posts/123');
await apiClient.post('/projects/create', data);
await apiClient.put('/users/update/456', updates);
await apiClient.get('/email/templates');

// ❌ 間違い - /api/ プレフィックスを追加しないでください
await apiClient.get('/api/blog/posts/123');  // 間違い!
await apiClient.post('/api/projects/create', data); // 間違い!
```

**マイクロサービスルーティング:**
- Form サービス: `/blog/*`
- Projects サービス: `/projects/*`
- Email サービス: `/email/*`
- Users サービス: `/users/*`

**理由:** API ルーティングはプロキシ設定で処理、`/api/` プレフィックス不要。

---

## Mutations

### 基本 Mutation パターン

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { myFeatureApi } from '../api/myFeatureApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const updateMutation = useMutation({
        mutationFn: (payload: UpdatePayload) =>
            myFeatureApi.updateEntity(blogId, entityId, payload),

        onSuccess: () => {
            // Invalidate して refetch
            queryClient.invalidateQueries({
                queryKey: ['entity', blogId, entityId]
            });
            showSuccess('Entity updated successfully');
        },

        onError: (error) => {
            showError('Failed to update entity');
            console.error('Update error:', error);
        },
    });

    const handleUpdate = () => {
        updateMutation.mutate({ name: 'New Name' });
    };

    return (
        <Button
            onClick={handleUpdate}
            disabled={updateMutation.isPending}
        >
            {updateMutation.isPending ? 'Updating...' : 'Update'}
        </Button>
    );
};
```

### Optimistic Updates

```typescript
const updateMutation = useMutation({
    mutationFn: (payload) => myFeatureApi.update(id, payload),

    // Optimistic update
    onMutate: async (newData) => {
        // 進行中の refetch をキャンセル
        await queryClient.cancelQueries({ queryKey: ['entity', id] });

        // 現在値のスナップショット
        const previousData = queryClient.getQueryData(['entity', id]);

        // Optimistically 更新
        queryClient.setQueryData(['entity', id], (old) => ({
            ...old,
            ...newData,
        }));

        // ロールバック関数を返却
        return { previousData };
    },

    // エラー時にロールバック
    onError: (err, newData, context) => {
        queryClient.setQueryData(['entity', id], context.previousData);
        showError('Update failed');
    },

    // 成功またはエラー後に refetch
    onSettled: () => {
        queryClient.invalidateQueries({ queryKey: ['entity', id] });
    },
});
```

---

## 高度なクエリパターン

### Prefetching

```typescript
export function usePrefetchEntity() {
    const queryClient = useQueryClient();

    return (blogId: number, entityId: number) => {
        return queryClient.prefetchQuery({
            queryKey: ['entity', blogId, entityId],
            queryFn: () => myFeatureApi.getEntity(blogId, entityId),
            staleTime: 5 * 60 * 1000,
        });
    };
}

// 使用: hover 時に prefetch
<div onMouseEnter={() => prefetch(blogId, id)}>
    <Link to={`/entity/${id}`}>View</Link>
</div>
```

### Fetch なしでキャッシュアクセス

```typescript
export function useEntityFromCache(blogId: number, entityId: number) {
    const queryClient = useQueryClient();

    // キャッシュから取得、なければ fetch しない
    const directCache = queryClient.getQueryData<MyEntity>(['entity', blogId, entityId]);

    if (directCache) return directCache;

    // グリッドキャッシュを試す
    const gridCache = queryClient.getQueryData<{ rows: MyEntity[] }>(['entities-v2', blogId]);

    return gridCache?.rows.find(row => row.id === entityId);
}
```

### 依存クエリ

```typescript
// ユーザーを先に fetch、その後ユーザー設定
const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => userApi.getUser(userId),
});

const { data: settings } = useSuspenseQuery({
    queryKey: ['user', userId, 'settings'],
    queryFn: () => settingsApi.getUserSettings(user.id),
    // Suspense により自動的に user ロード待機
});
```

---

## API クライアント設定

### apiClient 使用

```typescript
import apiClient from '@/lib/apiClient';

// apiClient は設定済み axios インスタンス
// 自動的に含む:
// - Base URL 設定
// - Cookie ベース認証
// - エラーインターセプター
// - レスポンストランスフォーマー
```

**新しい axios インスタンスを作成しない** - 一貫性のために apiClient を使用。

---

## クエリエラー処理

### onError コールバック

```typescript
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

const { showError } = useMuiSnackbar();

const { data } = useSuspenseQuery({
    queryKey: ['entity', id],
    queryFn: () => myFeatureApi.getEntity(id),

    // エラー処理
    onError: (error) => {
        showError('Failed to load entity');
        console.error('Load error:', error);
    },
});
```

### Error Boundaries

包括的なエラー処理のために Error Boundaries と結合:

```typescript
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
    fallback={<ErrorDisplay />}
    onError={(error) => console.error(error)}
>
    <SuspenseLoader>
        <ComponentWithSuspenseQuery />
    </SuspenseLoader>
</ErrorBoundary>
```

---

## 完全な例

### 例 1: 簡単な Entity Fetch

```typescript
import React from 'react';
import { useSuspenseQuery } from '@tanstack/react-query';
import { Box, Typography } from '@mui/material';
import { userApi } from '../api/userApi';

interface UserProfileProps {
    userId: string;
}

export const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
    const { data: user } = useSuspenseQuery({
        queryKey: ['user', userId],
        queryFn: () => userApi.getUser(userId),
        staleTime: 5 * 60 * 1000,
    });

    return (
        <Box>
            <Typography variant='h5'>{user.name}</Typography>
            <Typography>{user.email}</Typography>
        </Box>
    );
};

// Suspense と共に使用
<SuspenseLoader>
    <UserProfile userId='123' />
</SuspenseLoader>
```

### 例 2: Cache-First 戦略

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import type { Post } from '../types';

/**
 * Cache-first 戦略を持つ hook
 * API 呼び出し前にグリッドキャッシュを確認
 */
export function useSuspensePost(blogId: number, postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery<Post, Error>({
        queryKey: ['post', blogId, postId],
        queryFn: async () => {
            // 1. グリッドキャッシュを先に確認
            const gridCache = queryClient.getQueryData<{ rows: Post[] }>([
                'posts-v2',
                blogId,
                'summary'
            ]) || queryClient.getQueryData<{ rows: Post[] }>([
                'posts-v2',
                blogId,
                'flat'
            ]);

            if (gridCache?.rows) {
                const cached = gridCache.rows.find(row => row.S_ID === postId);
                if (cached) {
                    return cached;  // グリッドデータを再利用
                }
            }

            // 2. キャッシュにない、直接 fetch
            return postApi.getPost(blogId, postId);
        },
        staleTime: 5 * 60 * 1000,
        gcTime: 10 * 60 * 1000,
        refetchOnWindowFocus: false,
    });
}
```

**利点:**
- 重複 API 呼び出しを防止
- 既にロード済みなら即時データ
- キャッシュになければ API にフォールバック

### 例 3: 並列 Fetching

```typescript
import { useSuspenseQueries } from '@tanstack/react-query';

export const Dashboard: React.FC = () => {
    const [statsQuery, projectsQuery, notificationsQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['stats'],
                queryFn: () => statsApi.getStats(),
            },
            {
                queryKey: ['projects', 'active'],
                queryFn: () => projectsApi.getActiveProjects(),
            },
            {
                queryKey: ['notifications', 'unread'],
                queryFn: () => notificationsApi.getUnread(),
            },
        ],
    });

    return (
        <Box>
            <StatsCard data={statsQuery.data} />
            <ProjectsList projects={projectsQuery.data} />
            <Notifications items={notificationsQuery.data} />
        </Box>
    );
};
```

---

## キャッシュ Invalidation を持つ Mutations

### Update Mutation

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const useUpdatePost = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ blogId, postId, data }: UpdateParams) =>
            postApi.updatePost(blogId, postId, data),

        onSuccess: (data, variables) => {
            // 特定 post を invalidate
            queryClient.invalidateQueries({
                queryKey: ['post', variables.blogId, variables.postId]
            });

            // グリッド更新のためにリストを invalidate
            queryClient.invalidateQueries({
                queryKey: ['posts-v2', variables.blogId]
            });

            showSuccess('Post updated');
        },

        onError: (error) => {
            showError('Failed to update post');
            console.error('Update error:', error);
        },
    });
};

// 使用
const updatePost = useUpdatePost();

const handleSave = () => {
    updatePost.mutate({
        blogId: 123,
        postId: 456,
        data: { responses: { '101': 'value' } }
    });
};
```

### Delete Mutation

```typescript
export const useDeletePost = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ blogId, postId }: DeleteParams) =>
            postApi.deletePost(blogId, postId),

        onSuccess: (data, variables) => {
            // キャッシュから手動で削除 (optimistic)
            queryClient.setQueryData<{ rows: Post[] }>(
                ['posts-v2', variables.blogId],
                (old) => ({
                    ...old,
                    rows: old?.rows.filter(row => row.S_ID !== variables.postId) || []
                })
            );

            showSuccess('Post deleted');
        },

        onError: (error, variables) => {
            // ロールバック - 正確な状態のために refetch
            queryClient.invalidateQueries({
                queryKey: ['posts-v2', variables.blogId]
            });
            showError('Failed to delete post');
        },
    });
};
```

---

## クエリ設定ベストプラクティス

### デフォルト設定

```typescript
// QueryClientProvider 設定で
const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 1000 * 60 * 5,        // 5分
            gcTime: 1000 * 60 * 10,           // 10分 (以前の cacheTime)
            refetchOnWindowFocus: false,       // フォーカス時に refetch しない
            refetchOnMount: false,             // fresh ならマウント時に refetch しない
            retry: 1,                          // 失敗したクエリを1回リトライ
        },
    },
});
```

### クエリ別オーバーライド

```typescript
// 頻繁に変更されるデータ - 短い staleTime
useSuspenseQuery({
    queryKey: ['notifications', 'unread'],
    queryFn: () => notificationApi.getUnread(),
    staleTime: 30 * 1000,  // 30秒
});

// ほとんど変更されないデータ - 長い staleTime
useSuspenseQuery({
    queryKey: ['form', blogId, 'structure'],
    queryFn: () => formApi.getStructure(blogId),
    staleTime: 30 * 60 * 1000,  // 30分
});
```

---

## まとめ

**最新データフェッチングレシピ:**

1. **API サービス作成**: apiClient を使用する `features/X/api/XApi.ts`
2. **useSuspenseQuery 使用**: SuspenseLoader でラップしたコンポーネントで
3. **Cache-First**: API 呼び出し前にグリッドキャッシュを確認
4. **Query Keys**: 一貫したネーミング ['entity', id]
5. **Route 形式**: `/api/blog/route` ではなく `/blog/route`
6. **Mutations**: 成功後に invalidateQueries
7. **エラー処理**: onError + useMuiSnackbar
8. **型安全**: すべてのパラメータと戻り値を型付け

**参考:**
- [component-patterns.md](component-patterns.md) - Suspense 統合
- [loading-and-error-states.md](loading-and-error-states.md) - SuspenseLoader 使用
- [complete-examples.md](complete-examples.md) - 完全動作例
