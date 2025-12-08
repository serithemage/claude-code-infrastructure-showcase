# コンポーネントパターン

型安全、lazy loading、Suspense boundaries を強調するアプリケーションの最新 React コンポーネントアーキテクチャです。

---

## React.FC パターン (推奨)

### React.FC を使用する理由

すべてのコンポーネントは以下のために `React.FC<Props>` パターンを使用します:
- props に対する明示的な型安全
- 一貫したコンポーネントシグネチャ
- 明確な prop インターフェースドキュメント
- より良い IDE 自動補完

### 基本パターン

```typescript
import React from 'react';

interface MyComponentProps {
    /** 表示するユーザー ID */
    userId: number;
    /** アクション発生時の選択的コールバック */
    onAction?: () => void;
}

export const MyComponent: React.FC<MyComponentProps> = ({ userId, onAction }) => {
    return (
        <div>
            User: {userId}
        </div>
    );
};

export default MyComponent;
```

**キーポイント:**
- JSDoc コメント付きの別の Props インターフェース定義
- `React.FC<Props>` が型安全を提供
- パラメータで props 分割代入
- 下部に default export

---

## Lazy Loading パターン

### Lazy Load すべきとき

以下のようなコンポーネントを lazy load します:
- 重いもの (DataGrid、チャート、リッチテキストエディター)
- Route レベルコンポーネント
- Modal/dialog コンテンツ (初期に表示されない)
- Below-the-fold コンテンツ

### Lazy Load 方法

```typescript
import React from 'react';

// 重いコンポーネント lazy load
const PostDataGrid = React.lazy(() =>
    import('./grids/PostDataGrid')
);

// named exports の場合
const MyComponent = React.lazy(() =>
    import('./MyComponent').then(module => ({
        default: module.MyComponent
    }))
);
```

**PostTable.tsx の例:**

```typescript
/**
 * メイン post table コンテナコンポーネント
 */
import React, { useState, useCallback } from 'react';
import { Box, Paper } from '@mui/material';

// バンドルサイズ最適化のために PostDataGrid lazy load
const PostDataGrid = React.lazy(() => import('./grids/PostDataGrid'));

import { SuspenseLoader } from '~components/SuspenseLoader';

export const PostTable: React.FC<PostTableProps> = ({ formId }) => {
    return (
        <Box>
            <SuspenseLoader>
                <PostDataGrid formId={formId} />
            </SuspenseLoader>
        </Box>
    );
};

export default PostTable;
```

---

## Suspense Boundaries

### SuspenseLoader コンポーネント

**Import:**
```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
// または
import { SuspenseLoader } from '@/components/SuspenseLoader';
```

**使用:**
```typescript
<SuspenseLoader>
    <LazyLoadedComponent />
</SuspenseLoader>
```

**機能:**
- lazy コンポーネントロード中にローディングインジケーター表示
- スムーズなフェードインアニメーション
- 一貫したローディング体験
- レイアウトシフト防止

### Suspense Boundaries 配置場所

**Route レベル:**
```typescript
// routes/my-route/index.tsx
const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

function Route() {
    return (
        <SuspenseLoader>
            <MyPage />
        </SuspenseLoader>
    );
}
```

**コンポーネントレベル:**
```typescript
function ParentComponent() {
    return (
        <Box>
            <Header />
            <SuspenseLoader>
                <HeavyDataGrid />
            </SuspenseLoader>
        </Box>
    );
}
```

**複数 Boundaries:**
```typescript
function Page() {
    return (
        <Box>
            <SuspenseLoader>
                <HeaderSection />
            </SuspenseLoader>

            <SuspenseLoader>
                <MainContent />
            </SuspenseLoader>

            <SuspenseLoader>
                <Sidebar />
            </SuspenseLoader>
        </Box>
    );
}
```

各セクションが独立してロードされより良い UX。

---

## コンポーネント構造テンプレート

### 推奨順序

```typescript
/**
 * コンポーネント説明
 * 機能、使用タイミング
 */
import React, { useState, useCallback, useMemo, useEffect } from 'react';
import { Box, Paper, Button } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';

// Feature imports
import { myFeatureApi } from '../api/myFeatureApi';
import type { MyData } from '~types/myData';

// コンポーネント imports
import { SuspenseLoader } from '~components/SuspenseLoader';

// Hooks
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// 1. PROPS インターフェース (JSDoc と共に)
interface MyComponentProps {
    /** 表示する entity の ID */
    entityId: number;
    /** アクション完了時の選択的コールバック */
    onComplete?: () => void;
    /** 表示モード */
    mode?: 'view' | 'edit';
}

// 2. スタイル (インラインで100行未満の場合)
const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
    header: {
        mb: 2,
        display: 'flex',
        justifyContent: 'space-between',
    },
};

// 3. コンポーネント定義
export const MyComponent: React.FC<MyComponentProps> = ({
    entityId,
    onComplete,
    mode = 'view',
}) => {
    // 4. HOOKS (この順序で)
    // - Context hooks を先に
    const { user } = useAuth();
    const { showSuccess, showError } = useMuiSnackbar();

    // - データ fetching
    const { data } = useSuspenseQuery({
        queryKey: ['myEntity', entityId],
        queryFn: () => myFeatureApi.getEntity(entityId),
    });

    // - ローカル状態
    const [selectedItem, setSelectedItem] = useState<string | null>(null);
    const [isEditing, setIsEditing] = useState(mode === 'edit');

    // - Memoized 値
    const filteredData = useMemo(() => {
        return data.filter(item => item.active);
    }, [data]);

    // - Effects
    useEffect(() => {
        // セットアップ
        return () => {
            // クリーンアップ
        };
    }, []);

    // 5. イベントハンドラー (useCallback と共に)
    const handleItemSelect = useCallback((itemId: string) => {
        setSelectedItem(itemId);
    }, []);

    const handleSave = useCallback(async () => {
        try {
            await myFeatureApi.updateEntity(entityId, { /* data */ });
            showSuccess('Entity updated successfully');
            onComplete?.();
        } catch (error) {
            showError('Failed to update entity');
        }
    }, [entityId, onComplete, showSuccess, showError]);

    // 6. レンダー
    return (
        <Box sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <h2>My Component</h2>
                <Button onClick={handleSave}>Save</Button>
            </Box>

            <Paper sx={{ p: 2 }}>
                {filteredData.map(item => (
                    <div key={item.id}>{item.name}</div>
                ))}
            </Paper>
        </Box>
    );
};

// 7. EXPORT (下部に default export)
export default MyComponent;
```

---

## コンポーネント分離

### コンポーネントを分離すべきとき

**複数コンポーネントに分離すべきとき:**
- コンポーネントが300行超
- 複数の区別された責任
- 再利用可能なセクション
- 複雑なネスト JSX

**例:**

```typescript
// ❌ 避ける - モノリシック
function MassiveComponent() {
    // 500行以上
    // 検索ロジック
    // フィルターロジック
    // グリッドロジック
    // アクションパネルロジック
}

// ✅ 推奨 - モジュラー
function ParentContainer() {
    return (
        <Box>
            <SearchAndFilter onFilter={handleFilter} />
            <DataGrid data={filteredData} />
            <ActionPanel onAction={handleAction} />
        </Box>
    );
}
```

### 一緒に維持すべきとき

**同じファイルに維持すべきとき:**
- コンポーネント200行未満
- 密接に結合されたロジック
- 他で再利用不可
- 単純プレゼンテーションコンポーネント

---

## Export パターン

### Named Const + Default Export (推奨)

```typescript
export const MyComponent: React.FC<Props> = ({ ... }) => {
    // コンポーネントロジック
};

export default MyComponent;
```

**理由:**
- テスト/リファクタリングのための named export
- lazy loading 便宜のための default export
- 消費者に両方のオプション提供

### Named Exports Lazy Loading

```typescript
const MyComponent = React.lazy(() =>
    import('./MyComponent').then(module => ({
        default: module.MyComponent
    }))
);
```

---

## コンポーネント通信

### Props Down, Events Up

```typescript
// 親
function Parent() {
    const [selectedId, setSelectedId] = useState<string | null>(null);

    return (
        <Child
            data={data}                    // Props down
            onSelect={setSelectedId}       // Events up
        />
    );
}

// 子
interface ChildProps {
    data: Data[];
    onSelect: (id: string) => void;
}

export const Child: React.FC<ChildProps> = ({ data, onSelect }) => {
    return (
        <div onClick={() => onSelect(data[0].id)}>
            {/* コンテンツ */}
        </div>
    );
};
```

### Prop Drilling を避ける

**深いネストには context 使用:**
```typescript
// ❌ 避ける - 5段階以上 prop drilling
<A prop={x}>
  <B prop={x}>
    <C prop={x}>
      <D prop={x}>
        <E prop={x} />  // 結局ここで使用
      </D>
    </C>
  </B>
</A>

// ✅ 推奨 - Context または TanStack Query
const MyContext = createContext<MyData | null>(null);

function Provider({ children }) {
    const { data } = useSuspenseQuery({ ... });
    return <MyContext.Provider value={data}>{children}</MyContext.Provider>;
}

function DeepChild() {
    const data = useContext(MyContext);
    // 直接 data 使用
}
```

---

## 高度なパターン

### Compound Components

```typescript
// Card.tsx
export const Card: React.FC<CardProps> & {
    Header: typeof CardHeader;
    Body: typeof CardBody;
    Footer: typeof CardFooter;
} = ({ children }) => {
    return <Paper>{children}</Paper>;
};

Card.Header = CardHeader;
Card.Body = CardBody;
Card.Footer = CardFooter;

// 使用
<Card>
    <Card.Header>Title</Card.Header>
    <Card.Body>Content</Card.Body>
    <Card.Footer>Actions</Card.Footer>
</Card>
```

### Render Props (まれだが有用)

```typescript
interface DataProviderProps {
    children: (data: Data) => React.ReactNode;
}

export const DataProvider: React.FC<DataProviderProps> = ({ children }) => {
    const { data } = useSuspenseQuery({ ... });
    return <>{children(data)}</>;
};

// 使用
<DataProvider>
    {(data) => <Display data={data} />}
</DataProvider>
```

---

## まとめ

**最新コンポーネントレシピ:**
1. TypeScript と共に `React.FC<Props>`
2. 重ければ lazy load: `React.lazy(() => import())`
3. ローディングのために `<SuspenseLoader>` でラップ
4. データに `useSuspenseQuery` 使用
5. Import エイリアス (@/, ~types, ~components)
6. `useCallback` と共にイベントハンドラー
7. 下部に default export
8. ローディング状態での early returns 禁止

**参考:**
- [data-fetching.md](data-fetching.md) - useSuspenseQuery 詳細
- [loading-and-error-states.md](loading-and-error-states.md) - Suspense ベストプラクティス
- [complete-examples.md](complete-examples.md) - 完全動作例
