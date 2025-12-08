# ファイル構成

アプリケーションで保守可能でスケーラブルなフロントエンドコードのための正しいファイルおよびディレクトリ構造です。

---

## features/ vs components/ の区別

### features/ ディレクトリ

**目的**: 独自のロジック、API、コンポーネントを持つドメイン別機能

**使用タイミング:**
- 機能に複数の関連コンポーネントがあるとき
- 機能に独自の API エンドポイントがあるとき
- ドメイン別ロジックがあるとき
- カスタム hooks/ユーティリティがあるとき

**例:**
- `features/posts/` - プロジェクトカタログ/ポスト管理
- `features/blogs/` - ブログビルダーとレンダリング
- `features/auth/` - 認証フロー

**構造:**
```
features/
  my-feature/
    api/
      myFeatureApi.ts         # API サービスレイヤー
    components/
      MyFeatureMain.tsx       # メインコンポーネント
      SubComponents/          # 関連コンポーネント
    hooks/
      useMyFeature.ts         # カスタム hooks
      useSuspenseMyFeature.ts # Suspense hooks
    helpers/
      myFeatureHelpers.ts     # ユーティリティ関数
    types/
      index.ts                # TypeScript 型
    index.ts                  # Public exports
```

### components/ ディレクトリ

**目的**: 複数の機能で使用される真に再利用可能なコンポーネント

**使用タイミング:**
- コンポーネントが3箇所以上で使用されるとき
- コンポーネントがジェネリック (機能別ロジックなし)
- コンポーネントが UI プリミティブまたはパターンのとき

**例:**
- `components/SuspenseLoader/` - ローディングラッパー
- `components/CustomAppBar/` - アプリケーションヘッダー
- `components/ErrorBoundary/` - エラー処理
- `components/LoadingOverlay/` - ローディングオーバーレイ

**構造:**
```
components/
  SuspenseLoader/
    SuspenseLoader.tsx
    SuspenseLoader.test.tsx
  CustomAppBar/
    CustomAppBar.tsx
    CustomAppBar.test.tsx
```

---

## Feature ディレクトリ構造 (詳細)

### 完全な Feature 例

`features/posts/` 構造に基づく:

```
features/
  posts/
    api/
      postApi.ts              # API サービスレイヤー (GET, POST, PUT, DELETE)

    components/
      PostTable.tsx           # メインコンテナコンポーネント
      grids/
        PostDataGrid/
          PostDataGrid.tsx
      drawers/
        ProjectPostDrawer/
          ProjectPostDrawer.tsx
      cells/
        editors/
          TextEditCell.tsx
        renderers/
          DateCell.tsx
      toolbar/
        CustomToolbar.tsx

    hooks/
      usePostQueries.ts       # 一般クエリ
      useSuspensePost.ts      # Suspense クエリ
      usePostMutations.ts     # Mutations
      useGridLayout.ts        # 機能別 hooks

    helpers/
      postHelpers.ts          # ユーティリティ関数
      validation.ts           # バリデーションロジック

    types/
      index.ts                # TypeScript 型/インターフェース

    queries/
      postQueries.ts          # クエリキーファクトリ (オプション)

    context/
      PostContext.tsx         # React context (必要な場合)

    index.ts                  # Public API exports
```

### サブディレクトリガイドライン

#### api/ ディレクトリ

**目的**: 機能に対する中央化された API 呼び出し

**ファイル:**
- `{feature}Api.ts` - メイン API サービス

**パターン:**
```typescript
// features/my-feature/api/myFeatureApi.ts
import apiClient from '@/lib/apiClient';

export const myFeatureApi = {
    getItem: async (id: number) => {
        const { data } = await apiClient.get(`/blog/items/${id}`);
        return data;
    },
    createItem: async (payload) => {
        const { data } = await apiClient.post('/blog/items', payload);
        return data;
    },
};
```

#### components/ ディレクトリ

**目的**: 機能別コンポーネント

**構成:**
- 5個未満のコンポーネントならフラット構造
- 5個以上のコンポーネントなら責任別サブディレクトリ

**例:**
```
components/
  MyFeatureMain.tsx           # メインコンポーネント
  MyFeatureHeader.tsx         # サポートコンポーネント
  MyFeatureFooter.tsx

  # またはサブディレクトリ:
  containers/
    MyFeatureContainer.tsx
  presentational/
    MyFeatureDisplay.tsx
  blogs/
    MyFeatureBlog.tsx
```

#### hooks/ ディレクトリ

**目的**: 機能に対するカスタム hooks

**ネーミング:**
- `use` プレフィックス (camelCase)
- 機能を説明する名前

**例:**
```
hooks/
  useMyFeature.ts               # メイン hook
  useSuspenseMyFeature.ts       # Suspense バージョン
  useMyFeatureMutations.ts      # Mutations
  useMyFeatureFilters.ts        # フィルター/検索
```

#### helpers/ ディレクトリ

**目的**: 機能に特化したユーティリティ関数

**例:**
```
helpers/
  myFeatureHelpers.ts           # 一般ユーティリティ
  validation.ts                 # バリデーションロジック
  transformers.ts               # データ変換
  constants.ts                  # 定数
```

#### types/ ディレクトリ

**目的**: TypeScript 型定義

**ファイル:**
```
types/
  index.ts                      # メイン型、エクスポート
  internal.ts                   # 内部型 (エクスポートしない)
```

---

## Import エイリアス (Vite 設定)

### 利用可能なエイリアス

`vite.config.ts` 180-185行参照:

| エイリアス | 解決先 | 用途 |
|-------|-------------|---------|
| `@/` | `src/` | src ルートからの絶対パス import |
| `~types` | `src/types` | 共有 TypeScript 型 |
| `~components` | `src/components` | 再利用可能コンポーネント |
| `~features` | `src/features` | Feature imports |

### 使用例

```typescript
// ✅ 推奨 - 絶対パス import にエイリアス使用
import { apiClient } from '@/lib/apiClient';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { postApi } from '~features/posts/api/postApi';
import type { User } from '~types/user';

// ❌ 避ける - 深いネストでの相対パス
import { apiClient } from '../../../lib/apiClient';
import { SuspenseLoader } from '../../../components/SuspenseLoader';
```

### どのエイリアスを使うか

**@/ (一般)**:
- Lib ユーティリティ: `@/lib/apiClient`
- Hooks: `@/hooks/useAuth`
- Config: `@/config/theme`
- 共有サービス: `@/services/authService`

**~types (型 Import)**:
```typescript
import type { Post } from '~types/post';
import type { User, UserRole } from '~types/user';
```

**~components (再利用可能コンポーネント)**:
```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
import { CustomAppBar } from '~components/CustomAppBar';
import { ErrorBoundary } from '~components/ErrorBoundary';
```

**~features (Feature Imports)**:
```typescript
import { postApi } from '~features/posts/api/postApi';
import { useAuth } from '~features/auth/hooks/useAuth';
```

---

## ファイルネーミング規則

### コンポーネント

**パターン**: PascalCase + `.tsx` 拡張子

```
MyComponent.tsx
PostDataGrid.tsx
CustomAppBar.tsx
```

**避ける:**
- camelCase: `myComponent.tsx` ❌
- kebab-case: `my-component.tsx` ❌
- すべて大文字: `MYCOMPONENT.tsx` ❌

### Hooks

**パターン**: camelCase + `use` プレフィックス + `.ts` 拡張子

```
useMyFeature.ts
useSuspensePost.ts
useAuth.ts
useGridLayout.ts
```

### API サービス

**パターン**: camelCase + `Api` サフィックス + `.ts` 拡張子

```
myFeatureApi.ts
postApi.ts
userApi.ts
```

### Helpers/ユーティリティ

**パターン**: camelCase + 説明的な名前 + `.ts` 拡張子

```
myFeatureHelpers.ts
validation.ts
transformers.ts
constants.ts
```

### 型

**パターン**: camelCase、`index.ts` または説明的な名前

```
types/index.ts
types/post.ts
types/user.ts
```

---

## 新 Feature 作成タイミング

### 新 Feature 作成:

- 複数の関連コンポーネント (3個以上)
- 独自の API エンドポイントがある
- ドメイン別ロジック
- 時間とともに成長する
- 複数の routes で再利用

**例:** `features/posts/`
- 20個以上コンポーネント
- 独自 API サービス
- 複雑な状態管理
- 複数の routes で使用

### 既存 Feature に追加:

- 既存機能と関連
- 同じ API 共有
- 論理的にグループ化
- 既存機能を拡張

**例:** posts feature に export dialog 追加

### 再利用可能コンポーネント作成:

- 3個以上の機能で使用
- ジェネリック、ドメインロジックなし
- 純粋プレゼンテーション
- 共有パターン

**例:** `components/SuspenseLoader/`

---

## Import 構成

### Import 順序 (推奨)

```typescript
// 1. React および React 関連
import React, { useState, useCallback, useMemo } from 'react';
import { lazy } from 'react';

// 2. サードパーティライブラリ (アルファベット順)
import { Box, Paper, Button, Grid } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { createFileRoute } from '@tanstack/react-router';

// 3. エイリアス imports (@ を先に、次に ~)
import { apiClient } from '@/lib/apiClient';
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { postApi } from '~features/posts/api/postApi';

// 4. 型 imports (グループ化)
import type { Post } from '~types/post';
import type { User } from '~types/user';

// 5. 相対パス imports (同じ feature)
import { MySubComponent } from './MySubComponent';
import { useMyFeature } from '../hooks/useMyFeature';
import { myFeatureHelpers } from '../helpers/myFeatureHelpers';
```

すべての imports に **シングルクォート** 使用 (プロジェクト標準)

---

## Public API パターン

### feature/index.ts

クリーンな imports のために feature から public API エクスポート:

```typescript
// features/my-feature/index.ts

// メインコンポーネントエクスポート
export { MyFeatureMain } from './components/MyFeatureMain';
export { MyFeatureHeader } from './components/MyFeatureHeader';

// Hooks エクスポート
export { useMyFeature } from './hooks/useMyFeature';
export { useSuspenseMyFeature } from './hooks/useSuspenseMyFeature';

// API エクスポート
export { myFeatureApi } from './api/myFeatureApi';

// 型エクスポート
export type { MyFeatureData, MyFeatureConfig } from './types';
```

**使用:**
```typescript
// ✅ feature index からクリーンな import
import { MyFeatureMain, useMyFeature } from '~features/my-feature';

// ❌ 深い imports は避ける (必要なら OK)
import { MyFeatureMain } from '~features/my-feature/components/MyFeatureMain';
```

---

## ディレクトリ構造視覚化

```
src/
├── features/                    # ドメイン別機能
│   ├── posts/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── helpers/
│   │   ├── types/
│   │   └── index.ts
│   ├── blogs/
│   └── auth/
│
├── components/                  # 再利用可能コンポーネント
│   ├── SuspenseLoader/
│   ├── CustomAppBar/
│   ├── ErrorBoundary/
│   └── LoadingOverlay/
│
├── routes/                      # TanStack Router routes
│   ├── __root.tsx
│   ├── index.tsx
│   ├── project-catalog/
│   │   ├── index.tsx
│   │   └── create/
│   └── blogs/
│
├── hooks/                       # 共有 hooks
│   ├── useAuth.ts
│   ├── useMuiSnackbar.ts
│   └── useDebounce.ts
│
├── lib/                         # 共有ユーティリティ
│   ├── apiClient.ts
│   └── utils.ts
│
├── types/                       # 共有 TypeScript 型
│   ├── user.ts
│   ├── post.ts
│   └── common.ts
│
├── config/                      # 設定
│   └── theme.ts
│
└── App.tsx                      # ルートコンポーネント
```

---

## まとめ

**核心原則:**
1. **features/** ドメイン別コード用
2. **components/** 真に再利用可能な UI 用
3. サブディレクトリ使用: api/, components/, hooks/, helpers/, types/
4. クリーンな imports のための Import エイリアス (@/, ~types, ~components, ~features)
5. 一貫したネーミング: PascalCase コンポーネント、camelCase ユーティリティ
6. feature index.ts で public API エクスポート

**参考:**
- [component-patterns.md](component-patterns.md) - コンポーネント構造
- [data-fetching.md](data-fetching.md) - API サービスパターン
- [complete-examples.md](complete-examples.md) - 完全 feature 例
