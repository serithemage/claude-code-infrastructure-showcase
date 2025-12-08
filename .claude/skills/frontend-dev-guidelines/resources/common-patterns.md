# 共通パターン

forms、認証、DataGrid、dialogs およびその他一般的な UI 要素でよく使用されるパターンです。

---

## useAuth による認証

### 現在のユーザー取得

```typescript
import { useAuth } from '@/hooks/useAuth';

export const MyComponent: React.FC = () => {
    const { user } = useAuth();

    // 使用可能なプロパティ:
    // - user.id: string
    // - user.email: string
    // - user.username: string
    // - user.roles: string[]

    return (
        <div>
            <p>Logged in as: {user.email}</p>
            <p>Username: {user.username}</p>
            <p>Roles: {user.roles.join(', ')}</p>
        </div>
    );
};
```

**認証のために直接 API 呼び出ししないでください** - 常に `useAuth` hook を使用してください。

---

## React Hook Form を使用した Forms

### 基本 Form

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { TextField, Button } from '@mui/material';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// 検証のための Zod スキーマ
const formSchema = z.object({
    username: z.string().min(3, 'Username must be at least 3 characters'),
    email: z.string().email('Invalid email address'),
    age: z.number().min(18, 'Must be 18 or older'),
});

type FormData = z.infer<typeof formSchema>;

export const MyForm: React.FC = () => {
    const { showSuccess, showError } = useMuiSnackbar();

    const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
        resolver: zodResolver(formSchema),
        defaultValues: {
            username: '',
            email: '',
            age: 18,
        },
    });

    const onSubmit = async (data: FormData) => {
        try {
            await api.submitForm(data);
            showSuccess('Form submitted successfully');
        } catch (error) {
            showError('Failed to submit form');
        }
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <TextField
                {...register('username')}
                label='Username'
                error={!!errors.username}
                helperText={errors.username?.message}
            />

            <TextField
                {...register('email')}
                label='Email'
                error={!!errors.email}
                helperText={errors.email?.message}
                type='email'
            />

            <TextField
                {...register('age', { valueAsNumber: true })}
                label='Age'
                error={!!errors.age}
                helperText={errors.age?.message}
                type='number'
            />

            <Button type='submit' variant='contained'>
                Submit
            </Button>
        </form>
    );
};
```

---

## Dialog コンポーネントパターン

### 標準 Dialog 構造

BEST_PRACTICES.md より - すべての dialogs は以下を含む必要があります:
- タイトルにアイコン
- 閉じるボタン (X)
- 下部にアクションボタン

```typescript
import { Dialog, DialogTitle, DialogContent, DialogActions, Button, IconButton } from '@mui/material';
import { Close, Info } from '@mui/icons-material';

interface MyDialogProps {
    open: boolean;
    onClose: () => void;
    onConfirm: () => void;
}

export const MyDialog: React.FC<MyDialogProps> = ({ open, onClose, onConfirm }) => {
    return (
        <Dialog open={open} onClose={onClose} maxWidth='sm' fullWidth>
            <DialogTitle>
                <Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between' }}>
                    <Box sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
                        <Info color='primary' />
                        Dialog Title
                    </Box>
                    <IconButton onClick={onClose} size='small'>
                        <Close />
                    </IconButton>
                </Box>
            </DialogTitle>

            <DialogContent>
                {/* コンテンツ */}
            </DialogContent>

            <DialogActions>
                <Button onClick={onClose}>Cancel</Button>
                <Button onClick={onConfirm} variant='contained'>
                    Confirm
                </Button>
            </DialogActions>
        </Dialog>
    );
};
```

---

## DataGrid Wrapper パターン

### Wrapper コンポーネント契約

BEST_PRACTICES.md より - DataGrid wrappers は以下を受け取る必要があります:

**必須 Props:**
- `rows`: データ配列
- `columns`: カラム定義
- Loading/error 状態

**選択的 Props:**
- Toolbar コンポーネント
- カスタムアクション
- 初期状態

```typescript
import { DataGridPro } from '@mui/x-data-grid-pro';
import type { GridColDef } from '@mui/x-data-grid-pro';

interface DataGridWrapperProps {
    rows: any[];
    columns: GridColDef[];
    loading?: boolean;
    toolbar?: React.ReactNode;
    onRowClick?: (row: any) => void;
}

export const DataGridWrapper: React.FC<DataGridWrapperProps> = ({
    rows,
    columns,
    loading = false,
    toolbar,
    onRowClick,
}) => {
    return (
        <DataGridPro
            rows={rows}
            columns={columns}
            loading={loading}
            slots={{ toolbar: toolbar ? () => toolbar : undefined }}
            onRowClick={(params) => onRowClick?.(params.row)}
            // 標準設定
            pagination
            pageSizeOptions={[25, 50, 100]}
            initialState={{
                pagination: { paginationModel: { pageSize: 25 } },
            }}
        />
    );
};
```

---

## Mutation パターン

### キャッシュ Invalidation がある Update

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const useUpdateEntity = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ id, data }: { id: number; data: any }) =>
            api.updateEntity(id, data),

        onSuccess: (result, variables) => {
            // 影響を受けるクエリを invalidate
            queryClient.invalidateQueries({ queryKey: ['entity', variables.id] });
            queryClient.invalidateQueries({ queryKey: ['entities'] });

            showSuccess('Entity updated');
        },

        onError: () => {
            showError('Failed to update entity');
        },
    });
};

// 使用
const updateEntity = useUpdateEntity();

const handleSave = () => {
    updateEntity.mutate({ id: 123, data: { name: 'New Name' } });
};
```

---

## 状態管理パターン

### サーバー状態のための TanStack Query (主要)

**すべてのサーバーデータ**に TanStack Query 使用:
- Fetching: useSuspenseQuery
- Mutations: useMutation
- キャッシング: 自動
- 同期: 内蔵

```typescript
// ✅ 正しい - サーバーデータに TanStack Query
const { data: users } = useSuspenseQuery({
    queryKey: ['users'],
    queryFn: () => userApi.getUsers(),
});
```

### UI 状態のための useState

**ローカル UI 状態にのみ** `useState` 使用:
- Form 入力 (uncontrolled)
- Modal 開閉
- 選択されたタブ
- 一時的な UI フラグ

```typescript
// ✅ 正しい - UI 状態に useState
const [modalOpen, setModalOpen] = useState(false);
const [selectedTab, setSelectedTab] = useState(0);
```

### グローバルクライアント状態のための Zustand (最小限に)

**グローバルクライアント状態にのみ** Zustand 使用:
- テーマ設定
- サイドバー折りたたみ状態
- ユーザー設定 (サーバーから来ないもの)

```typescript
import { create } from 'zustand';

interface AppState {
    sidebarOpen: boolean;
    toggleSidebar: () => void;
}

export const useAppState = create<AppState>((set) => ({
    sidebarOpen: true,
    toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

**prop drilling 避ける** - 代わりに context か Zustand 使用。

---

## まとめ

**共通パターン:**
- ✅ 現在のユーザーのための useAuth hook (id, email, roles, username)
- ✅ forms のための React Hook Form + Zod
- ✅ アイコン + 閉じるボタンがある Dialog
- ✅ DataGrid wrapper 契約
- ✅ キャッシュ invalidation がある Mutations
- ✅ サーバー状態のための TanStack Query
- ✅ UI 状態のための useState
- ✅ グローバルクライアント状態のための Zustand (最小限に)

**参考:**
- [data-fetching.md](data-fetching.md) - TanStack Query パターン
- [component-patterns.md](component-patterns.md) - コンポーネント構造
- [loading-and-error-states.md](loading-and-error-states.md) - エラー処理
