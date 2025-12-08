# パフォーマンス最適化

React コンポーネントのパフォーマンス最適化、不要な再レンダリング防止、メモリリーク防止のためのパターンです。

---

## Memoization パターン

### 高コスト計算に useMemo 使用

```typescript
import { useMemo } from 'react';

export const DataDisplay: React.FC<{ items: Item[], searchTerm: string }> = ({
    items,
    searchTerm,
}) => {
    // ❌ 避けてください - 毎レンダーで実行
    const filteredItems = items
        .filter(item => item.name.includes(searchTerm))
        .sort((a, b) => a.name.localeCompare(b.name));

    // ✅ 正しい - Memoized、依存性変更時のみ再計算
    const filteredItems = useMemo(() => {
        return items
            .filter(item => item.name.toLowerCase().includes(searchTerm.toLowerCase()))
            .sort((a, b) => a.name.localeCompare(b.name));
    }, [items, searchTerm]);

    return <List items={filteredItems} />;
};
```

**useMemo を使用すべき時:**
- 大規模配列のフィルタリング/ソート
- 複雑な計算
- データ構造変換
- 高コスト演算 (ループ、再帰)

**useMemo を使用すべきでない時:**
- 単純な文字列連結
- 基本的な算術
- 早期最適化 (まずプロファイリングしてください！)

---

## イベントハンドラに useCallback 使用

### 問題

```typescript
// ❌ 避けてください - 毎レンダーで新しい関数生成
export const Parent: React.FC = () => {
    const handleClick = (id: string) => {
        console.log('Clicked:', id);
    };

    // Parent がレンダーされるたびに Child が再レンダー
    // handleClick が毎回新しい関数参照だから
    return <Child onClick={handleClick} />;
};
```

### 解決策

```typescript
import { useCallback } from 'react';

export const Parent: React.FC = () => {
    // ✅ 正しい - 安定した関数参照
    const handleClick = useCallback((id: string) => {
        console.log('Clicked:', id);
    }, []); // 空の deps = 関数が絶対変更されない

    // props が実際に変更された時のみ Child 再レンダー
    return <Child onClick={handleClick} />;
};
```

**useCallback を使用すべき時:**
- 子に props として渡される関数
- useEffect の依存性として使用される関数
- memoized コンポーネントに渡される関数
- リストのイベントハンドラ

**useCallback を使用すべきでない時:**
- 子に渡されないイベントハンドラ
- 単純なインラインハンドラ: `onClick={() => doSomething()}`

---

## コンポーネント Memoization に React.memo 使用

### 基本使用法

```typescript
import React from 'react';

interface ExpensiveComponentProps {
    data: ComplexData;
    onAction: () => void;
}

// ✅ 高コストコンポーネントを React.memo でラップ
export const ExpensiveComponent = React.memo<ExpensiveComponentProps>(
    function ExpensiveComponent({ data, onAction }) {
        // 複雑なレンダリングロジック
        return <ComplexVisualization data={data} />;
    }
);
```

**React.memo を使用すべき時:**
- 頻繁にレンダーされるコンポーネント
- 高コストレンダリングがあるコンポーネント
- props が頻繁に変更されない時
- リストアイテムコンポーネント
- DataGrid セル/レンダラー

**React.memo を使用すべきでない時:**
- props がどうせ頻繁に変更される時
- レンダリングが既に高速な時
- 早期最適化

---

## Debounced 検索

### use-debounce Hook 使用

```typescript
import { useState } from 'react';
import { useDebounce } from 'use-debounce';
import { useSuspenseQuery } from '@tanstack/react-query';

export const SearchComponent: React.FC = () => {
    const [searchTerm, setSearchTerm] = useState('');

    // 300ms 間 debounce
    const [debouncedSearchTerm] = useDebounce(searchTerm, 300);

    // クエリは debounced 値を使用
    const { data } = useSuspenseQuery({
        queryKey: ['search', debouncedSearchTerm],
        queryFn: () => api.search(debouncedSearchTerm),
        enabled: debouncedSearchTerm.length > 0,
    });

    return (
        <input
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            placeholder='Search...'
        />
    );
};
```

**最適な Debounce タイミング:**
- **300-500ms**: 検索/フィルタリング
- **1000ms**: 自動保存
- **100-200ms**: リアルタイム検証

---

## メモリリーク防止

### Timeout/Interval クリーンアップ

```typescript
import { useEffect, useState } from 'react';

export const MyComponent: React.FC = () => {
    const [count, setCount] = useState(0);

    useEffect(() => {
        // ✅ 正しい - Interval クリーンアップ
        const intervalId = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);

        return () => {
            clearInterval(intervalId);  // クリーンアップ！
        };
    }, []);

    useEffect(() => {
        // ✅ 正しい - Timeout クリーンアップ
        const timeoutId = setTimeout(() => {
            console.log('Delayed action');
        }, 5000);

        return () => {
            clearTimeout(timeoutId);  // クリーンアップ！
        };
    }, []);

    return <div>{count}</div>;
};
```

### イベントリスナークリーンアップ

```typescript
useEffect(() => {
    const handleResize = () => {
        console.log('Resized');
    };

    window.addEventListener('resize', handleResize);

    return () => {
        window.removeEventListener('resize', handleResize);  // クリーンアップ！
    };
}, []);
```

### Fetch に Abort Controllers 使用

```typescript
useEffect(() => {
    const abortController = new AbortController();

    fetch('/api/data', { signal: abortController.signal })
        .then(response => response.json())
        .then(data => setState(data))
        .catch(error => {
            if (error.name === 'AbortError') {
                console.log('Fetch aborted');
            }
        });

    return () => {
        abortController.abort();  // クリーンアップ！
    };
}, []);
```

**参考**: TanStack Query 使用時はこれが自動的に処理されます。

---

## フォームパフォーマンス

### 特定フィールドのみ Watch (全体ではなく)

```typescript
import { useForm } from 'react-hook-form';

export const MyForm: React.FC = () => {
    const { register, watch, handleSubmit } = useForm();

    // ❌ 避けてください - すべてのフィールドを watch、すべての変更で再レンダー
    const formValues = watch();

    // ✅ 正しい - 必要なものだけ watch
    const username = watch('username');
    const email = watch('email');

    // または複数の特定フィールド
    const [username, email] = watch(['username', 'email']);

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input {...register('username')} />
            <input {...register('email')} />
            <input {...register('password')} />

            {/* username/email 変更時のみ再レンダー */}
            <p>Username: {username}, Email: {email}</p>
        </form>
    );
};
```

---

## リストレンダリング最適化

### Key Prop 使用

```typescript
// ✅ 正しい - 安定した一意のキー
{items.map(item => (
    <ListItem key={item.id}>
        {item.name}
    </ListItem>
))}

// ❌ 避けてください - インデックスをキーに (リスト変更時に不安定)
{items.map((item, index) => (
    <ListItem key={index}>  // リスト順序変更時に問題
        {item.name}
    </ListItem>
))}
```

### Memoized リストアイテム

```typescript
const ListItem = React.memo<ListItemProps>(({ item, onAction }) => {
    return (
        <Box onClick={() => onAction(item.id)}>
            {item.name}
        </Box>
    );
});

export const List: React.FC<{ items: Item[] }> = ({ items }) => {
    const handleAction = useCallback((id: string) => {
        console.log('Action:', id);
    }, []);

    return (
        <Box>
            {items.map(item => (
                <ListItem
                    key={item.id}
                    item={item}
                    onAction={handleAction}
                />
            ))}
        </Box>
    );
};
```

---

## コンポーネント再初期化防止

### 問題

```typescript
// ❌ 避けてください - 毎レンダーでコンポーネント再生成
export const Parent: React.FC = () => {
    // 毎レンダーで新しいコンポーネント定義！
    const ChildComponent = () => <div>Child</div>;

    return <ChildComponent />;  // 毎レンダーでアンマウントしてリマウント
};
```

### 解決策

```typescript
// ✅ 正しい - 外部に定義するか useMemo 使用
const ChildComponent: React.FC = () => <div>Child</div>;

export const Parent: React.FC = () => {
    return <ChildComponent />;  // 安定したコンポーネント
};

// ✅ または動的な場合は useMemo 使用
export const Parent: React.FC<{ config: Config }> = ({ config }) => {
    const DynamicComponent = useMemo(() => {
        return () => <div>{config.title}</div>;
    }, [config.title]);

    return <DynamicComponent />;
};
```

---

## 重い依存関係の Lazy Loading

### コードスプリッティング

```typescript
// ❌ 避けてください - トップレベルで重いライブラリを import
import jsPDF from 'jspdf';  // 大きなライブラリが即座にロード
import * as XLSX from 'xlsx';  // 大きなライブラリが即座にロード

// ✅ 正しい - 必要な時に動的 import
const handleExportPDF = async () => {
    const { jsPDF } = await import('jspdf');
    const doc = new jsPDF();
    // 使用
};

const handleExportExcel = async () => {
    const XLSX = await import('xlsx');
    // 使用
};
```

---

## まとめ

**パフォーマンスチェックリスト:**
- ✅ 高コスト計算に `useMemo` (filter, sort, map)
- ✅ 子に渡される関数に `useCallback`
- ✅ 高コストコンポーネントに `React.memo`
- ✅ 検索/フィルター debounce (300-500ms)
- ✅ useEffect で timeout/interval クリーンアップ
- ✅ 特定フォームフィールドのみ watch (全体ではなく)
- ✅ リストに安定したキー
- ✅ 重いライブラリを lazy load
- ✅ React.lazy でコードスプリッティング

**参考:**
- [component-patterns.md](component-patterns.md) - Lazy loading
- [data-fetching.md](data-fetching.md) - TanStack Query 最適化
- [complete-examples.md](complete-examples.md) - コンテキスト内パフォーマンスパターン
