---
name: frontend-dev-guidelines
description: React/TypeScript アプリケーションのためのフロントエンド開発ガイドライン。Suspense、lazy loading、useSuspenseQuery、features ディレクトリを使用したファイル構成、MUI v7 styling、TanStack Router、パフォーマンス最適化、TypeScript ベストプラクティスを含む最新パターン。コンポーネント、ページ、機能作成、データfetching、styling、routing またはフロントエンドコード作業時に使用。
---

# フロントエンド開発ガイドライン

## 目的

Suspense ベースのデータfetching、lazy loading、適切なファイル構成、パフォーマンス最適化を強調する最新React開発のための総合ガイドです。

## このSkill使用タイミング

- 新コンポーネントまたはページ作成
- 新機能構築
- TanStack Queryでのデータfetching
- TanStack Routerでのrouting設定
- MUI v7でのコンポーネントstyling
- パフォーマンス最適化
- フロントエンドコード構成
- TypeScript ベストプラクティス

---

## クイックスタート

### 新コンポーネントチェックリスト

コンポーネントを作成しますか？このチェックリストに従ってください：

- [ ] TypeScriptと共に `React.FC<Props>` パターン使用
- [ ] 重いコンポーネントの場合 Lazy load: `React.lazy(() => import())`
- [ ] Loading 状態のために `<SuspenseLoader>` でラップ
- [ ] データfetchingに `useSuspenseQuery` 使用
- [ ] Import aliases: `@/`, `~types`, `~components`, `~features`
- [ ] スタイル: 100行未満ならインライン、100行超なら別ファイル
- [ ] 子に渡されるイベントハンドラーに `useCallback` 使用
- [ ] 下部に default export
- [ ] Loading スピナーを使用した early return 禁止
- [ ] ユーザー通知に `useMuiSnackbar` 使用

### 新機能チェックリスト

機能を作成しますか？この構造をセットアップしてください：

- [ ] `features/{feature-name}/` ディレクトリ作成
- [ ] サブディレクトリ作成: `api/`, `components/`, `hooks/`, `helpers/`, `types/`
- [ ] API service ファイル作成: `api/{feature}Api.ts`
- [ ] `types/` に TypeScript 型設定
- [ ] `routes/{feature-name}/index.tsx` に route 作成
- [ ] 機能コンポーネント Lazy load
- [ ] Suspense boundaries 使用
- [ ] 機能 `index.ts` で public API export

---

## Import Aliases クイックリファレンス

| Alias | 解決先 | 例 |
|-------|-------------|---------|
| `@/` | `src/` | `import { apiClient } from '@/lib/apiClient'` |
| `~types` | `src/types` | `import type { User } from '~types/user'` |
| `~components` | `src/components` | `import { SuspenseLoader } from '~components/SuspenseLoader'` |
| `~features` | `src/features` | `import { authApi } from '~features/auth'` |

定義場所: [vite.config.ts](../../vite.config.ts) 180-185行

---

## 共通 Imports チートシート

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

## トピックガイド

### 🎨 コンポーネントパターン

**最新 React コンポーネント使用:**
- 型安全のための `React.FC<Props>`
- コード分割のための `React.lazy()`
- Loading 状態のための `SuspenseLoader`
- Named const + default export パターン

**核心概念:**
- 重いコンポーネント Lazy load (DataGrid、チャート、エディター)
- Lazy コンポーネントは常に Suspense でラップ
- SuspenseLoader コンポーネント使用 (fade アニメーション含む)
- コンポーネント構造: Props → Hooks → Handlers → Render → Export

**[📖 完全ガイド: resources/component-patterns.md](resources/component-patterns.md)**

---

### 📊 データ Fetching

**基本パターン: useSuspenseQuery**
- Suspense boundaries と共に使用
- Cache-first 戦略 (API 前に grid キャッシュ確認)
- `isLoading` チェック代替
- ジェネリックで型安全

**API Service レイヤー:**
- `features/{feature}/api/{feature}Api.ts` 作成
- `apiClient` axios インスタンス使用
- 機能別中央化されたメソッド
- Route 形式: `/form/route` (`/api/form/route` ではない)

**[📖 完全ガイド: resources/data-fetching.md](resources/data-fetching.md)**

---

### 📁 ファイル構成

**features/ vs components/:**
- `features/`: ドメイン特化 (posts, comments, auth)
- `components/`: 真に再利用可能なもの (SuspenseLoader, CustomAppBar)

**Feature サブディレクトリ:**
```
features/
  my-feature/
    api/          # API service レイヤー
    components/   # 機能コンポーネント
    hooks/        # Custom hooks
    helpers/      # ユーティリティ関数
    types/        # TypeScript 型
```

**[📖 完全ガイド: resources/file-organization.md](resources/file-organization.md)**

---

### 🎨 Styling

**インライン vs 分離:**
- 100行未満: インライン `const styles: Record<string, SxProps<Theme>>`
- 100行超: 別の `.styles.ts` ファイル

**基本方法:**
- MUI コンポーネントに `sx` prop 使用
- `SxProps<Theme>` で型安全
- Theme アクセス: `(theme) => theme.palette.primary.main`

**MUI v7 Grid:**
```typescript
<Grid size={{ xs: 12, md: 6 }}>  // ✅ v7 文法
<Grid xs={12} md={6}>             // ❌ 以前の文法
```

**[📖 完全ガイド: resources/styling-guide.md](resources/styling-guide.md)**

---

### 🛣️ Routing

**TanStack Router - フォルダベース:**
- ディレクトリ: `routes/my-route/index.tsx`
- コンポーネント Lazy load
- `createFileRoute` 使用
- Loader に Breadcrumb データ

**例:**
```typescript
import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

export const Route = createFileRoute('/my-route/')({
    component: MyPage,
    loader: () => ({ crumb: 'My Route' }),
});
```

**[📖 完全ガイド: resources/routing-guide.md](resources/routing-guide.md)**

---

### ⏳ Loading & Error 状態

**核心ルール: Early Return 禁止**

```typescript
// ❌ 絶対ダメ - レイアウトシフト誘発
if (isLoading) {
    return <LoadingSpinner />;
}

// ✅ 常に - 一貫したレイアウト
<SuspenseLoader>
    <Content />
</SuspenseLoader>
```

**理由:** Cumulative Layout Shift (CLS) 防止、より良い UX

**Error Handling:**
- ユーザーフィードバックに `useMuiSnackbar` 使用
- `react-toastify` 絶対使用禁止
- TanStack Query `onError` コールバック

**[📖 完全ガイド: resources/loading-and-error-states.md](resources/loading-and-error-states.md)**

---

### ⚡ パフォーマンス

**最適化パターン:**
- `useMemo`: コストの高い計算 (filter, sort, map)
- `useCallback`: 子に渡されるイベントハンドラー
- `React.memo`: コストの高いコンポーネント
- Debounced 検索 (300-500ms)
- メモリリーク防止 (useEffect で cleanup)

**[📖 完全ガイド: resources/performance.md](resources/performance.md)**

---

### 📘 TypeScript

**標準:**
- Strict モード、`any` 型禁止
- 関数に明示的戻り値型
- Type imports: `import type { User } from '~types/user'`
- JSDoc が含まれたコンポーネント prop インターフェース

**[📖 完全ガイド: resources/typescript-standards.md](resources/typescript-standards.md)**

---

### 🔧 共通パターン

**カバーするトピック:**
- Zod 検証と React Hook Form
- DataGrid wrapper 契約
- Dialog コンポーネント標準
- 現在ユーザーのための `useAuth` hook
- キャッシュ無効化を含む Mutation パターン

**[📖 完全ガイド: resources/common-patterns.md](resources/common-patterns.md)**

---

### 📚 完全例

**動作する完全例:**
- すべてのパターンが含まれた最新コンポーネント
- 完全な機能構造
- API service レイヤー
- Lazy loading が含まれた Route
- Suspense + useSuspenseQuery
- 検証が含まれた Form

**[📖 完全ガイド: resources/complete-examples.md](resources/complete-examples.md)**

---

## ナビゲーションガイド

| 必要な作業... | 読むべきリソース |
|------------|-------------------|
| コンポーネント作成 | [component-patterns.md](resources/component-patterns.md) |
| データ fetch | [data-fetching.md](resources/data-fetching.md) |
| ファイル/フォルダ構成 | [file-organization.md](resources/file-organization.md) |
| コンポーネントスタイリング | [styling-guide.md](resources/styling-guide.md) |
| Routing 設定 | [routing-guide.md](resources/routing-guide.md) |
| Loading/errors 処理 | [loading-and-error-states.md](resources/loading-and-error-states.md) |
| パフォーマンス最適化 | [performance.md](resources/performance.md) |
| TypeScript 型 | [typescript-standards.md](resources/typescript-standards.md) |
| Forms/Auth/DataGrid | [common-patterns.md](resources/common-patterns.md) |
| 完全例を見る | [complete-examples.md](resources/complete-examples.md) |

---

## 核心原則

1. **重いものはすべて Lazy Load**: Routes, DataGrid, チャート, エディター
2. **Loading に Suspense**: early return の代わりに SuspenseLoader 使用
3. **useSuspenseQuery**: 新コードのデフォルトデータ fetching パターン
4. **機能は整理される**: api/, components/, hooks/, helpers/ サブディレクトリ
5. **サイズに応じたスタイル**: 100行未満インライン、100行超分離
6. **Import Aliases**: @/, ~types, ~components, ~features 使用
7. **Early Return 禁止**: レイアウトシフト防止
8. **useMuiSnackbar**: すべてのユーザー通知に使用

---

## クイックリファレンス: ファイル構造

```
src/
  features/
    my-feature/
      api/
        myFeatureApi.ts       # API service
      components/
        MyFeature.tsx         # メインコンポーネント
        SubComponent.tsx      # 関連コンポーネント
      hooks/
        useMyFeature.ts       # Custom hooks
        useSuspenseMyFeature.ts  # Suspense hooks
      helpers/
        myFeatureHelpers.ts   # ユーティリティ
      types/
        index.ts              # TypeScript 型
      index.ts                # Public exports

  components/
    SuspenseLoader/
      SuspenseLoader.tsx      # 再利用可能な loader
    CustomAppBar/
      CustomAppBar.tsx        # 再利用可能な app bar

  routes/
    my-route/
      index.tsx               # Route コンポーネント
      create/
        index.tsx             # ネストされた route
```

---

## 最新コンポーネントテンプレート (クイックコピー)

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

完全例は [resources/complete-examples.md](resources/complete-examples.md) を参照

---

## 関連 Skills

- **error-tracking**: Sentry を使用した error tracking (フロントエンドにも適用)
- **backend-dev-guidelines**: フロントエンドが消費するバックエンド API パターン

---

**Skill 状態**: 最適な context 管理のための progressive loading が含まれたモジュラー構造
