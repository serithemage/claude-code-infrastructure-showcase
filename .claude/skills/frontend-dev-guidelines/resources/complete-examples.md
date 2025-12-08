# 完全な例

React.FC、lazy loading、Suspense、useSuspenseQuery、スタイリング、ルーティング、エラー処理など、すべての最新パターンを組み合わせた完全な動作例です。

---

## 例 1: 完全な最新コンポーネント

組み合わせ: React.FC、useSuspenseQuery、cache-first、useCallback、スタイリング、エラー処理

```typescript
/**
 * ユーザープロファイル表示コンポーネント
 * Suspense と TanStack Query を使用した最新パターンのデモ
 */
import React, { useState, useCallback, useMemo } from 'react';
import { Box, Paper, Typography, Button, Avatar } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { userApi } from '../api/userApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import type { User } from '~types/user';

// スタイルオブジェクト
const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 3,
        maxWidth: 600,
        margin: '0 auto',
    },
    header: {
        display: 'flex',
        alignItems: 'center',
        gap: 2,
        mb: 3,
    },
    content: {
        display: 'flex',
        flexDirection: 'column',
        gap: 2,
    },
    actions: {
        display: 'flex',
        gap: 1,
        mt: 2,
    },
};

interface UserProfileProps {
    userId: string;
    onUpdate?: () => void;
}

export const UserProfile: React.FC<UserProfileProps> = ({ userId, onUpdate }) => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();
    const [isEditing, setIsEditing] = useState(false);

    // Suspense クエリ - isLoading 不要！
    const { data: user } = useSuspenseQuery({
        queryKey: ['user', userId],
        queryFn: () => userApi.getUser(userId),
        staleTime: 5 * 60 * 1000,
    });

    // Update mutation
    const updateMutation = useMutation({
        mutationFn: (updates: Partial<User>) =>
            userApi.updateUser(userId, updates),

        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['user', userId] });
            showSuccess('Profile updated');
            setIsEditing(false);
            onUpdate?.();
        },

        onError: () => {
            showError('Failed to update profile');
        },
    });

    // Memoized 計算値
    const fullName = useMemo(() => {
        return `${user.firstName} ${user.lastName}`;
    }, [user.firstName, user.lastName]);

    // useCallback を使用したイベントハンドラ
    const handleEdit = useCallback(() => {
        setIsEditing(true);
    }, []);

    const handleSave = useCallback(() => {
        updateMutation.mutate({
            firstName: user.firstName,
            lastName: user.lastName,
        });
    }, [user, updateMutation]);

    const handleCancel = useCallback(() => {
        setIsEditing(false);
    }, []);

    return (
        <Paper sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <Avatar sx={{ width: 64, height: 64 }}>
                    {user.firstName[0]}{user.lastName[0]}
                </Avatar>
                <Box>
                    <Typography variant='h5'>{fullName}</Typography>
                    <Typography color='text.secondary'>{user.email}</Typography>
                </Box>
            </Box>

            <Box sx={componentStyles.content}>
                <Typography>Username: {user.username}</Typography>
                <Typography>Roles: {user.roles.join(', ')}</Typography>
            </Box>

            <Box sx={componentStyles.actions}>
                {!isEditing ? (
                    <Button variant='contained' onClick={handleEdit}>
                        Edit Profile
                    </Button>
                ) : (
                    <>
                        <Button
                            variant='contained'
                            onClick={handleSave}
                            disabled={updateMutation.isPending}
                        >
                            {updateMutation.isPending ? 'Saving...' : 'Save'}
                        </Button>
                        <Button onClick={handleCancel}>
                            Cancel
                        </Button>
                    </>
                )}
            </Box>
        </Paper>
    );
};

export default UserProfile;
```

**使用法:**
```typescript
<SuspenseLoader>
    <UserProfile userId='123' onUpdate={() => console.log('Updated')} />
</SuspenseLoader>
```

---

## 例 2: 完全な Feature 構造

`features/posts/` ベースの実際の例:

```
features/
  users/
    api/
      userApi.ts                # API サービスレイヤー
    components/
      UserProfile.tsx           # メインコンポーネント (例 1 より)
      UserList.tsx              # リストコンポーネント
      UserForm.tsx              # フォームコンポーネント
      modals/
        DeleteUserModal.tsx     # モーダルコンポーネント
    hooks/
      useSuspenseUser.ts        # Suspense クエリ hook
      useUserMutations.ts       # Mutation hooks
      useUserPermissions.ts     # Feature 別 hook
    helpers/
      userHelpers.ts            # ユーティリティ関数
      validation.ts             # 検証ロジック
    types/
      index.ts                  # TypeScript インターフェース
    index.ts                    # Public API exports
```

### API サービス (userApi.ts)

```typescript
import apiClient from '@/lib/apiClient';
import type { User, CreateUserPayload, UpdateUserPayload } from '../types';

export const userApi = {
    getUser: async (userId: string): Promise<User> => {
        const { data } = await apiClient.get(`/users/${userId}`);
        return data;
    },

    getUsers: async (): Promise<User[]> => {
        const { data } = await apiClient.get('/users');
        return data;
    },

    createUser: async (payload: CreateUserPayload): Promise<User> => {
        const { data } = await apiClient.post('/users', payload);
        return data;
    },

    updateUser: async (userId: string, payload: UpdateUserPayload): Promise<User> => {
        const { data } = await apiClient.put(`/users/${userId}`, payload);
        return data;
    },

    deleteUser: async (userId: string): Promise<void> => {
        await apiClient.delete(`/users/${userId}`);
    },
};
```

### Suspense Hook (useSuspenseUser.ts)

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { userApi } from '../api/userApi';
import type { User } from '../types';

export function useSuspenseUser(userId: string) {
    return useSuspenseQuery<User, Error>({
        queryKey: ['user', userId],
        queryFn: () => userApi.getUser(userId),
        staleTime: 5 * 60 * 1000,
        gcTime: 10 * 60 * 1000,
    });
}

export function useSuspenseUsers() {
    return useSuspenseQuery<User[], Error>({
        queryKey: ['users'],
        queryFn: () => userApi.getUsers(),
        staleTime: 1 * 60 * 1000,  // リストにはより短く
    });
}
```

### 型 (types/index.ts)

```typescript
export interface User {
    id: string;
    username: string;
    email: string;
    firstName: string;
    lastName: string;
    roles: string[];
    createdAt: string;
    updatedAt: string;
}

export interface CreateUserPayload {
    username: string;
    email: string;
    firstName: string;
    lastName: string;
    password: string;
}

export type UpdateUserPayload = Partial<Omit<User, 'id' | 'createdAt' | 'updatedAt'>>;
```

### Public Exports (index.ts)

```typescript
// コンポーネント export
export { UserProfile } from './components/UserProfile';
export { UserList } from './components/UserList';

// Hooks export
export { useSuspenseUser, useSuspenseUsers } from './hooks/useSuspenseUser';
export { useUserMutations } from './hooks/useUserMutations';

// API export
export { userApi } from './api/userApi';

// 型 export
export type { User, CreateUserPayload, UpdateUserPayload } from './types';
```

---

## 例 3: Lazy Loading がある完全な Route

```typescript
/**
 * ユーザープロファイル route
 * パス: /users/:userId
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';
import { SuspenseLoader } from '~components/SuspenseLoader';

// UserProfile コンポーネント lazy load
const UserProfile = lazy(() =>
    import('@/features/users/components/UserProfile').then(
        (module) => ({ default: module.UserProfile })
    )
);

export const Route = createFileRoute('/users/$userId')({
    component: UserProfilePage,
    loader: ({ params }) => ({
        crumb: `User ${params.userId}`,
    }),
});

function UserProfilePage() {
    const { userId } = Route.useParams();

    return (
        <SuspenseLoader>
            <UserProfile
                userId={userId}
                onUpdate={() => console.log('Profile updated')}
            />
        </SuspenseLoader>
    );
}

export default UserProfilePage;
```

---

## 例 4: 検索とフィルタリングがあるリスト

```typescript
import React, { useState, useMemo } from 'react';
import { Box, TextField, List, ListItem } from '@mui/material';
import { useDebounce } from 'use-debounce';
import { useSuspenseQuery } from '@tanstack/react-query';
import { userApi } from '../api/userApi';

export const UserList: React.FC = () => {
    const [searchTerm, setSearchTerm] = useState('');
    const [debouncedSearch] = useDebounce(searchTerm, 300);

    const { data: users } = useSuspenseQuery({
        queryKey: ['users'],
        queryFn: () => userApi.getUsers(),
    });

    // Memoized フィルタリング
    const filteredUsers = useMemo(() => {
        if (!debouncedSearch) return users;

        return users.filter(user =>
            user.name.toLowerCase().includes(debouncedSearch.toLowerCase()) ||
            user.email.toLowerCase().includes(debouncedSearch.toLowerCase())
        );
    }, [users, debouncedSearch]);

    return (
        <Box>
            <TextField
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
                placeholder='Search users...'
                fullWidth
                sx={{ mb: 2 }}
            />

            <List>
                {filteredUsers.map(user => (
                    <ListItem key={user.id}>
                        {user.name} - {user.email}
                    </ListItem>
                ))}
            </List>
        </Box>
    );
};
```

---

## 例 5: 検証がある Form

```typescript
import React from 'react';
import { Box, TextField, Button, Paper } from '@mui/material';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { userApi } from '../api/userApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

const userSchema = z.object({
    username: z.string().min(3).max(50),
    email: z.string().email(),
    firstName: z.string().min(1),
    lastName: z.string().min(1),
});

type UserFormData = z.infer<typeof userSchema>;

interface CreateUserFormProps {
    onSuccess?: () => void;
}

export const CreateUserForm: React.FC<CreateUserFormProps> = ({ onSuccess }) => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const { register, handleSubmit, formState: { errors }, reset } = useForm<UserFormData>({
        resolver: zodResolver(userSchema),
        defaultValues: {
            username: '',
            email: '',
            firstName: '',
            lastName: '',
        },
    });

    const createMutation = useMutation({
        mutationFn: (data: UserFormData) => userApi.createUser(data),

        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['users'] });
            showSuccess('User created successfully');
            reset();
            onSuccess?.();
        },

        onError: () => {
            showError('Failed to create user');
        },
    });

    const onSubmit = (data: UserFormData) => {
        createMutation.mutate(data);
    };

    return (
        <Paper sx={{ p: 3, maxWidth: 500 }}>
            <form onSubmit={handleSubmit(onSubmit)}>
                <Box sx={{ display: 'flex', flexDirection: 'column', gap: 2 }}>
                    <TextField
                        {...register('username')}
                        label='Username'
                        error={!!errors.username}
                        helperText={errors.username?.message}
                        fullWidth
                    />

                    <TextField
                        {...register('email')}
                        label='Email'
                        type='email'
                        error={!!errors.email}
                        helperText={errors.email?.message}
                        fullWidth
                    />

                    <TextField
                        {...register('firstName')}
                        label='First Name'
                        error={!!errors.firstName}
                        helperText={errors.firstName?.message}
                        fullWidth
                    />

                    <TextField
                        {...register('lastName')}
                        label='Last Name'
                        error={!!errors.lastName}
                        helperText={errors.lastName?.message}
                        fullWidth
                    />

                    <Button
                        type='submit'
                        variant='contained'
                        disabled={createMutation.isPending}
                    >
                        {createMutation.isPending ? 'Creating...' : 'Create User'}
                    </Button>
                </Box>
            </form>
        </Paper>
    );
};

export default CreateUserForm;
```

---

## 例 6: Lazy Loading がある親コンテナ

```typescript
import React from 'react';
import { Box } from '@mui/material';
import { SuspenseLoader } from '~components/SuspenseLoader';

// 重いコンポーネント lazy load
const UserList = React.lazy(() => import('./UserList'));
const UserStats = React.lazy(() => import('./UserStats'));
const ActivityFeed = React.lazy(() => import('./ActivityFeed'));

export const UserDashboard: React.FC = () => {
    return (
        <Box sx={{ p: 2 }}>
            <SuspenseLoader>
                <UserStats />
            </SuspenseLoader>

            <Box sx={{ display: 'flex', gap: 2, mt: 2 }}>
                <Box sx={{ flex: 2 }}>
                    <SuspenseLoader>
                        <UserList />
                    </SuspenseLoader>
                </Box>

                <Box sx={{ flex: 1 }}>
                    <SuspenseLoader>
                        <ActivityFeed />
                    </SuspenseLoader>
                </Box>
            </Box>
        </Box>
    );
};

export default UserDashboard;
```

**利点:**
- 各セクションが独立してロード
- ユーザーが部分コンテンツをより早く見れる
- より良い体感パフォーマンス

---

## 例 7: Cache-First 戦略実装

useSuspensePost.ts ベースの完全な例:

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import type { Post } from '../types';

/**
 * Cache-first 戦略があるスマート post hook
 * 可能な場合グリッドキャッシュのデータ再利用
 */
export function useSuspensePost(blogId: number, postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery<Post, Error>({
        queryKey: ['post', blogId, postId],
        queryFn: async () => {
            // 戦略 1: グリッドキャッシュをまず確認 (API 呼び出し防止)
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
                const cached = gridCache.rows.find(
                    (row) => row.S_ID === postId
                );

                if (cached) {
                    return cached;  // キャッシュから返す - API 呼び出しなし！
                }
            }

            // 戦略 2: キャッシュにない、API から fetch
            return postApi.getPost(blogId, postId);
        },
        staleTime: 5 * 60 * 1000,       // 5分間 fresh
        gcTime: 10 * 60 * 1000,          // 10分間キャッシュ
        refetchOnWindowFocus: false,     // フォーカス時 refetch しない
    });
}
```

**このパターンを使用する理由:**
- API 前にグリッドキャッシュ確認
- ユーザーがグリッドから来たら即座にデータ
- キャッシュにない場合は API へ fallback
- 設定可能なキャッシュ時間

---

## 例 8: 完全な Route ファイル

```typescript
/**
 * Project catalog route
 * パス: /project-catalog
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

// PostTable コンポーネント lazy load
const PostTable = lazy(() =>
    import('@/features/posts/components/PostTable').then(
        (module) => ({ default: module.PostTable })
    )
);

// Route 定数
const PROJECT_CATALOG_FORM_ID = 744;
const PROJECT_CATALOG_PROJECT_ID = 225;

export const Route = createFileRoute('/project-catalog/')({
    component: ProjectCatalogPage,
    loader: () => ({
        crumb: 'Projects',  // Breadcrumb タイトル
    }),
});

function ProjectCatalogPage() {
    return (
        <PostTable
            blogId={PROJECT_CATALOG_FORM_ID}
            projectId={PROJECT_CATALOG_PROJECT_ID}
            tableType='active_projects'
            title='Form Dashboard'
        />
    );
}

export default ProjectCatalogPage;
```

---

## 例 9: Form がある Dialog

```typescript
import React from 'react';
import {
    Dialog,
    DialogTitle,
    DialogContent,
    DialogActions,
    Button,
    TextField,
    Box,
    IconButton,
} from '@mui/material';
import { Close, PersonAdd } from '@mui/icons-material';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const formSchema = z.object({
    name: z.string().min(1),
    email: z.string().email(),
});

type FormData = z.infer<typeof formSchema>;

interface AddUserDialogProps {
    open: boolean;
    onClose: () => void;
    onSubmit: (data: FormData) => Promise<void>;
}

export const AddUserDialog: React.FC<AddUserDialogProps> = ({
    open,
    onClose,
    onSubmit,
}) => {
    const { register, handleSubmit, formState: { errors }, reset } = useForm<FormData>({
        resolver: zodResolver(formSchema),
    });

    const handleClose = () => {
        reset();
        onClose();
    };

    const handleFormSubmit = async (data: FormData) => {
        await onSubmit(data);
        handleClose();
    };

    return (
        <Dialog open={open} onClose={handleClose} maxWidth='sm' fullWidth>
            <DialogTitle>
                <Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between' }}>
                    <Box sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
                        <PersonAdd color='primary' />
                        Add User
                    </Box>
                    <IconButton onClick={handleClose} size='small'>
                        <Close />
                    </IconButton>
                </Box>
            </DialogTitle>

            <form onSubmit={handleSubmit(handleFormSubmit)}>
                <DialogContent>
                    <Box sx={{ display: 'flex', flexDirection: 'column', gap: 2 }}>
                        <TextField
                            {...register('name')}
                            label='Name'
                            error={!!errors.name}
                            helperText={errors.name?.message}
                            fullWidth
                            autoFocus
                        />

                        <TextField
                            {...register('email')}
                            label='Email'
                            type='email'
                            error={!!errors.email}
                            helperText={errors.email?.message}
                            fullWidth
                        />
                    </Box>
                </DialogContent>

                <DialogActions>
                    <Button onClick={handleClose}>Cancel</Button>
                    <Button type='submit' variant='contained'>
                        Add User
                    </Button>
                </DialogActions>
            </form>
        </Dialog>
    );
};
```

---

## 例 10: 並列データ Fetching

```typescript
import React from 'react';
import { Box, Grid, Paper } from '@mui/material';
import { useSuspenseQueries } from '@tanstack/react-query';
import { userApi } from '../api/userApi';
import { statsApi } from '../api/statsApi';
import { activityApi } from '../api/activityApi';

export const Dashboard: React.FC = () => {
    // Suspense ですべてのデータを並列で fetch
    const [statsQuery, usersQuery, activityQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['stats'],
                queryFn: () => statsApi.getStats(),
            },
            {
                queryKey: ['users', 'active'],
                queryFn: () => userApi.getActiveUsers(),
            },
            {
                queryKey: ['activity', 'recent'],
                queryFn: () => activityApi.getRecent(),
            },
        ],
    });

    return (
        <Box sx={{ p: 2 }}>
            <Grid container spacing={2}>
                <Grid size={{ xs: 12, md: 4 }}>
                    <Paper sx={{ p: 2 }}>
                        <h3>Stats</h3>
                        <p>Total: {statsQuery.data.total}</p>
                    </Paper>
                </Grid>

                <Grid size={{ xs: 12, md: 4 }}>
                    <Paper sx={{ p: 2 }}>
                        <h3>Active Users</h3>
                        <p>Count: {usersQuery.data.length}</p>
                    </Paper>
                </Grid>

                <Grid size={{ xs: 12, md: 4 }}>
                    <Paper sx={{ p: 2 }}>
                        <h3>Recent Activity</h3>
                        <p>Events: {activityQuery.data.length}</p>
                    </Paper>
                </Grid>
            </Grid>
        </Box>
    );
};

// Suspense と一緒に使用
<SuspenseLoader>
    <Dashboard />
</SuspenseLoader>
```

---

## 例 11: Optimistic Update

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import type { User } from '../types';

export const useToggleUserStatus = () => {
    const queryClient = useQueryClient();

    return useMutation({
        mutationFn: (userId: string) => userApi.toggleStatus(userId),

        // Optimistic update
        onMutate: async (userId) => {
            // 進行中の refetch キャンセル
            await queryClient.cancelQueries({ queryKey: ['users'] });

            // 以前の値のスナップショット
            const previousUsers = queryClient.getQueryData<User[]>(['users']);

            // Optimistically UI 更新
            queryClient.setQueryData<User[]>(['users'], (old) => {
                return old?.map(user =>
                    user.id === userId
                        ? { ...user, active: !user.active }
                        : user
                ) || [];
            });

            return { previousUsers };
        },

        // エラー時ロールバック
        onError: (err, userId, context) => {
            queryClient.setQueryData(['users'], context?.previousUsers);
        },

        // mutation 後 refetch
        onSettled: () => {
            queryClient.invalidateQueries({ queryKey: ['users'] });
        },
    });
};
```

---

## まとめ

**核心ポイント:**

1. **コンポーネントパターン**: React.FC + lazy + Suspense + useSuspenseQuery
2. **Feature 構造**: 構成されたサブディレクトリ (api/, components/, hooks/ など)
3. **ルーティング**: Lazy loading があるフォルダベース
4. **データ Fetching**: Cache-first 戦略がある useSuspenseQuery
5. **Forms**: React Hook Form + Zod 検証
6. **エラー処理**: useMuiSnackbar + onError コールバック
7. **パフォーマンス**: useMemo、useCallback、React.memo、debouncing
8. **スタイリング**: 100行未満インライン、sx prop、MUI v7 文法

**各パターンの詳細説明は他のリソース参照。**
