# スタイリングガイド

MUI v7 sx prop、インラインスタイル、テーマ統合を使用した最新スタイリングパターンです。

---

## インライン vs 別ファイルスタイル

### 判断基準

**<100行: コンポーネント上部にインラインスタイル**

```typescript
import type { SxProps, Theme } from '@mui/material';

const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
    header: {
        mb: 2,
        borderBottom: '1px solid',
        borderColor: 'divider',
    },
    // ... 更にスタイル
};

export const MyComponent: React.FC = () => {
    return (
        <Box sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <h2>Title</h2>
            </Box>
        </Box>
    );
};
```

**>100行: 別の `.styles.ts` ファイル**

```typescript
// MyComponent.styles.ts
import type { SxProps, Theme } from '@mui/material';

export const componentStyles: Record<string, SxProps<Theme>> = {
    container: { ... },
    header: { ... },
    // ... 100行以上のスタイル
};

// MyComponent.tsx
import { componentStyles } from './MyComponent.styles';

export const MyComponent: React.FC = () => {
    return <Box sx={componentStyles.container}>...</Box>;
};
```

### 実例: UnifiedForm.tsx

**48-126行**: 78行のインラインスタイル (許容)

```typescript
const formStyles: Record<string, SxProps<Theme>> = {
    gridContainer: {
        height: '100%',
        maxHeight: 'calc(100vh - 220px)',
    },
    section: {
        height: '100%',
        maxHeight: 'calc(100vh - 220px)',
        overflow: 'auto',
        p: 4,
    },
    // ... 15個以上のスタイルオブジェクト
};
```

**ガイドライン**: ユーザーは ~80行インラインが快適。100行付近で判断して決定。

---

## sx Prop パターン

### 基本使用法

```typescript
<Box sx={{ p: 2, mb: 3, display: 'flex' }}>
    Content
</Box>
```

### テーマアクセスと共に

```typescript
<Box
    sx={{
        p: 2,
        backgroundColor: (theme) => theme.palette.primary.main,
        color: (theme) => theme.palette.primary.contrastText,
        borderRadius: (theme) => theme.shape.borderRadius,
    }}
>
    Themed Box
</Box>
```

### レスポンシブスタイル

```typescript
<Box
    sx={{
        p: { xs: 1, sm: 2, md: 3 },
        width: { xs: '100%', md: '50%' },
        flexDirection: { xs: 'column', md: 'row' },
    }}
>
    Responsive Layout
</Box>
```

### Pseudo-Selectors

```typescript
<Box
    sx={{
        p: 2,
        '&:hover': {
            backgroundColor: 'rgba(0,0,0,0.05)',
        },
        '&:active': {
            backgroundColor: 'rgba(0,0,0,0.1)',
        },
        '& .child-class': {
            color: 'primary.main',
        },
    }}
>
    Interactive Box
</Box>
```

---

## MUI v7 パターン

### Grid コンポーネント (v7 文法)

```typescript
import { Grid } from '@mui/material';

// ✅ 正しい - size prop がある v7 文法
<Grid container spacing={2}>
    <Grid size={{ xs: 12, md: 6 }}>
        Left Column
    </Grid>
    <Grid size={{ xs: 12, md: 6 }}>
        Right Column
    </Grid>
</Grid>

// ❌ 間違い - 古い v6 文法
<Grid container spacing={2}>
    <Grid xs={12} md={6}>  {/* 古い - 使用禁止 */}
        Content
    </Grid>
</Grid>
```

**核心変更**: `xs={12} md={6}` の代わりに `size={{ xs: 12, md: 6 }}`

### レスポンシブ Grid

```typescript
<Grid container spacing={3}>
    <Grid size={{ xs: 12, sm: 6, md: 4, lg: 3 }}>
        Responsive Column
    </Grid>
</Grid>
```

### ネスト Grids

```typescript
<Grid container spacing={2}>
    <Grid size={{ xs: 12, md: 8 }}>
        <Grid container spacing={1}>
            <Grid size={{ xs: 12, sm: 6 }}>
                Nested 1
            </Grid>
            <Grid size={{ xs: 12, sm: 6 }}>
                Nested 2
            </Grid>
        </Grid>
    </Grid>

    <Grid size={{ xs: 12, md: 4 }}>
        Sidebar
    </Grid>
</Grid>
```

---

## 型安全スタイル

### スタイルオブジェクト型

```typescript
import type { SxProps, Theme } from '@mui/material';

// 型安全スタイル
const styles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        // 自動補完と型チェックがここで動作
    },
};

// または個別スタイル
const containerStyle: SxProps<Theme> = {
    p: 2,
    display: 'flex',
};
```

### テーマ認識スタイル

```typescript
const styles: Record<string, SxProps<Theme>> = {
    primary: {
        color: (theme) => theme.palette.primary.main,
        backgroundColor: (theme) => theme.palette.primary.light,
        '&:hover': {
            backgroundColor: (theme) => theme.palette.primary.dark,
        },
    },
    customSpacing: {
        padding: (theme) => theme.spacing(2),
        margin: (theme) => theme.spacing(1, 2), // 上/下: 1, 左/右: 2
    },
};
```

---

## 使用すべきでないもの

### ❌ makeStyles (MUI v4 パターン)

```typescript
// ❌ 避ける - 古い Material-UI v4 パターン
import { makeStyles } from '@mui/styles';

const useStyles = makeStyles((theme) => ({
    root: {
        padding: theme.spacing(2),
    },
}));
```

**避けるべき理由**: deprecated、v7 で十分にサポートされていない

### ❌ styled() コンポーネント

```typescript
// ❌ 避ける - styled-components パターン
import { styled } from '@mui/material/styles';

const StyledBox = styled(Box)(({ theme }) => ({
    padding: theme.spacing(2),
}));
```

**避けるべき理由**: sx prop がより柔軟で新コンポーネントを生成しない

### ✅ 代わりに sx Prop 使用

```typescript
// ✅ 推奨
<Box
    sx={{
        p: 2,
        backgroundColor: 'primary.main',
    }}
>
    Content
</Box>
```

---

## コードスタイル標準

### インデント

**4 スペース** (2ではなく、タブではなく)

```typescript
const styles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
};
```

### クォート

**シングルクォート** 文字列用 (プロジェクト標準)

```typescript
// ✅ 正しい
const color = 'primary.main';
import { Box } from '@mui/material';

// ❌ 間違い
const color = "primary.main";
import { Box } from "@mui/material";
```

### 末尾カンマ

オブジェクトと配列に**常に末尾カンマ**使用

```typescript
// ✅ 正しい
const styles = {
    container: { p: 2 },
    header: { mb: 1 },  // 末尾カンマ
};

const items = [
    'item1',
    'item2',  // 末尾カンマ
];

// ❌ 間違い - 末尾カンマなし
const styles = {
    container: { p: 2 },
    header: { mb: 1 }  // カンマ欠落
};
```

---

## 一般的なスタイルパターン

### Flexbox レイアウト

```typescript
const styles = {
    flexRow: {
        display: 'flex',
        flexDirection: 'row',
        alignItems: 'center',
        gap: 2,
    },
    flexColumn: {
        display: 'flex',
        flexDirection: 'column',
        gap: 1,
    },
    spaceBetween: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
    },
};
```

### スペーシング

```typescript
// Padding
p: 2           // 全面
px: 2          // 水平 (左 + 右)
py: 2          // 垂直 (上 + 下)
pt: 2, pr: 1   // 特定面

// Margin
m: 2, mx: 2, my: 2, mt: 2, mr: 1

// 単位: 1 = 8px (theme.spacing(1))
p: 2  // = 16px
p: 0.5  // = 4px
```

### ポジショニング

```typescript
const styles = {
    relative: {
        position: 'relative',
    },
    absolute: {
        position: 'absolute',
        top: 0,
        right: 0,
    },
    sticky: {
        position: 'sticky',
        top: 0,
        zIndex: 1000,
    },
};
```

---

## まとめ

**スタイリングチェックリスト:**
- ✅ MUI スタイリングに `sx` prop 使用
- ✅ `SxProps<Theme>` で型安全
- ✅ <100行: インライン; >100行: 別ファイル
- ✅ MUI v7 Grid: `size={{ xs: 12 }}`
- ✅ 4 スペースインデント
- ✅ シングルクォート
- ✅ 末尾カンマ
- ❌ makeStyles や styled() 使用禁止

**参考:**
- [component-patterns.md](component-patterns.md) - コンポーネント構造
- [complete-examples.md](complete-examples.md) - 完全スタイリング例
