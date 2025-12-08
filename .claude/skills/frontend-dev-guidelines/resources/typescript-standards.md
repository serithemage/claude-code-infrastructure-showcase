# TypeScript 標準

React フロントエンドコードの型安全と保守性のための TypeScript ベストプラクティスです。

---

## Strict モード

### 設定

プロジェクトで TypeScript strict モードが**有効化**されています:

```json
// tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "noImplicitAny": true,
        "strictNullChecks": true
    }
}
```

**意味:**
- 暗黙的な `any` 型禁止
- Null/undefined を明示的に処理する必要あり
- 型安全を強制

---

## `any` 型禁止

### ルール

```typescript
// ❌ 絶対 any 使用禁止
function handleData(data: any) {
    return data.something;
}

// ✅ 特定の型使用
interface MyData {
    something: string;
}

function handleData(data: MyData) {
    return data.something;
}

// ✅ または本当に不明なデータには unknown 使用
function handleUnknown(data: unknown) {
    if (typeof data === 'object' && data !== null && 'something' in data) {
        return (data as MyData).something;
    }
}
```

**型が本当に分からない場合:**
- `unknown` 使用 (型チェック強制)
- Type guards で絞り込む
- 型が不明な理由をドキュメント化

---

## 明示的戻り値型

### 関数戻り値型

```typescript
// ✅ 正しい - 明示的戻り値型
function getUser(id: number): Promise<User> {
    return apiClient.get(`/users/${id}`);
}

function calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
}

// ❌ 避ける - 暗黙的戻り値型 (不明確)
function getUser(id: number) {
    return apiClient.get(`/users/${id}`);
}
```

### コンポーネント戻り値型

```typescript
// React.FC が既に戻り値型提供 (ReactElement)
export const MyComponent: React.FC<Props> = ({ prop }) => {
    return <div>{prop}</div>;
};

// カスタム hooks の場合
function useMyData(id: number): { data: Data; isLoading: boolean } {
    const [data, setData] = useState<Data | null>(null);
    const [isLoading, setIsLoading] = useState(true);

    return { data: data!, isLoading };
}
```

---

## 型 Imports

### 'type' キーワード使用

```typescript
// ✅ 正しい - 型 import 明示的に表示
import type { User } from '~types/user';
import type { Post } from '~types/post';
import type { SxProps, Theme } from '@mui/material';

// ❌ 避ける - 値と型 import 混合
import { User } from '~types/user';  // 型か値か不明確
```

**利点:**
- 型と値を明確に分離
- より良い tree-shaking
- 循環依存防止
- TypeScript コンパイラ最適化

---

## コンポーネント Prop インターフェース

### インターフェースパターン

```typescript
/**
 * MyComponent の Props
 */
interface MyComponentProps {
    /** 表示するユーザー ID */
    userId: number;

    /** アクション完了時の選択的コールバック */
    onComplete?: () => void;

    /** コンポーネント表示モード */
    mode?: 'view' | 'edit';

    /** 追加 CSS クラス */
    className?: string;
}

export const MyComponent: React.FC<MyComponentProps> = ({
    userId,
    onComplete,
    mode = 'view',  // デフォルト値
    className,
}) => {
    return <div>...</div>;
};
```

**キーポイント:**
- props のための別インターフェース
- 各 prop に JSDoc コメント
- 選択的 props は `?` 使用
- 分割代入でデフォルト値提供

### Children 付き Props

```typescript
interface ContainerProps {
    children: React.ReactNode;
    title: string;
}

// React.FC は自動的に children 型含むが、明示的に記述
export const Container: React.FC<ContainerProps> = ({ children, title }) => {
    return (
        <div>
            <h2>{title}</h2>
            {children}
        </div>
    );
};
```

---

## ユーティリティ型

### Partial<T>

```typescript
// すべてのプロパティを選択的に
type UserUpdate = Partial<User>;

function updateUser(id: number, updates: Partial<User>) {
    // updates は User プロパティの任意の部分集合可能
}
```

### Pick<T, K>

```typescript
// 特定プロパティ選択
type UserPreview = Pick<User, 'id' | 'name' | 'email'>;

const preview: UserPreview = {
    id: 1,
    name: 'John',
    email: 'john@example.com',
    // 他の User プロパティ許可されない
};
```

### Omit<T, K>

```typescript
// 特定プロパティ除外
type UserWithoutPassword = Omit<User, 'password' | 'passwordHash'>;

const publicUser: UserWithoutPassword = {
    id: 1,
    name: 'John',
    email: 'john@example.com',
    // password と passwordHash 許可されない
};
```

### Required<T>

```typescript
// すべてのプロパティを必須に
type RequiredConfig = Required<Config>;  // すべての選択的 props が必須に
```

### Record<K, V>

```typescript
// 型安全オブジェクト/マップ
const userMap: Record<string, User> = {
    'user1': { id: 1, name: 'John' },
    'user2': { id: 2, name: 'Jane' },
};

// スタイル用
import type { SxProps, Theme } from '@mui/material';

const styles: Record<string, SxProps<Theme>> = {
    container: { p: 2 },
    header: { mb: 1 },
};
```

---

## Type Guards

### 基本 Type Guards

```typescript
function isUser(data: unknown): data is User {
    return (
        typeof data === 'object' &&
        data !== null &&
        'id' in data &&
        'name' in data
    );
}

// 使用
if (isUser(response)) {
    console.log(response.name);  // TypeScript が User として認識
}
```

### Discriminated Unions

```typescript
type LoadingState =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: Data }
    | { status: 'error'; error: Error };

function Component({ state }: { state: LoadingState }) {
    // TypeScript が status に基づいて型絞り込み
    if (state.status === 'success') {
        return <Display data={state.data} />;  // ここで data 使用可能
    }

    if (state.status === 'error') {
        return <Error error={state.error} />;  // ここで error 使用可能
    }

    return <Loading />;
}
```

---

## ジェネリック型

### ジェネリック関数

```typescript
function getById<T>(items: T[], id: number): T | undefined {
    return items.find(item => (item as any).id === id);
}

// 型推論と共に使用
const users: User[] = [...];
const user = getById(users, 123);  // 型: User | undefined
```

### ジェネリックコンポーネント

```typescript
interface ListProps<T> {
    items: T[];
    renderItem: (item: T) => React.ReactNode;
}

export function List<T>({ items, renderItem }: ListProps<T>): React.ReactElement {
    return (
        <div>
            {items.map((item, index) => (
                <div key={index}>{renderItem(item)}</div>
            ))}
        </div>
    );
}

// 使用
<List<User>
    items={users}
    renderItem={(user) => <UserCard user={user} />}
/>
```

---

## 型アサーション (慎重に使用)

### 使用すべきとき

```typescript
// ✅ OK - TypeScript より詳しく知っているとき
const element = document.getElementById('my-element') as HTMLInputElement;
const value = element.value;

// ✅ OK - 検証した API レスポンス
const response = await api.getData();
const user = response.data as User;  // 形状を知っている
```

### 使用すべきでないとき

```typescript
// ❌ 避ける - 型安全をバイパス
const data = getData() as any;  // 間違い - TypeScript 無効化

// ❌ 避ける - 安全でないアサーション
const value = unknownValue as string;  // 実際に string でないかもしれない
```

---

## Null/Undefined 処理

### Optional Chaining

```typescript
// ✅ 正しい
const name = user?.profile?.name;

// 同等のコード:
const name = user && user.profile && user.profile.name;
```

### Nullish Coalescing

```typescript
// ✅ 正しい
const displayName = user?.name ?? 'Anonymous';

// null または undefined のときのみデフォルト値使用
// (|| と異なる - '' や 0、false ではトリガーしない)
```

### Non-Null Assertion (慎重に使用)

```typescript
// ✅ OK - 値が存在すると確信しているとき
const data = queryClient.getQueryData<Data>(['data'])!;

// ⚠️ 注意 - null でないと分かっているときのみ使用
// 明示的チェックがより良い:
const data = queryClient.getQueryData<Data>(['data']);
if (data) {
    // data 使用
}
```

---

## まとめ

**TypeScript チェックリスト:**
- ✅ Strict モード有効化
- ✅ `any` 型禁止 (必要なら `unknown` 使用)
- ✅ 関数に明示的戻り値型
- ✅ 型 imports に `import type` 使用
- ✅ prop インターフェースに JSDoc コメント
- ✅ ユーティリティ型 (Partial, Pick, Omit, Required, Record)
- ✅ 絞り込みのための Type guards
- ✅ Optional chaining と nullish coalescing
- ❌ どうしても必要な場合以外は型アサーション避ける

**参考:**
- [component-patterns.md](component-patterns.md) - コンポーネント型付け
- [data-fetching.md](data-fetching.md) - API 型付け
