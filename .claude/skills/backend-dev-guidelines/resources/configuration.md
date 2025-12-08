# 設定管理 - UnifiedConfigパターン

バックエンドマイクロサービスの設定管理の完全なガイドです。

## 目次

- [UnifiedConfig概要](#unifiedconfig概要)
- [process.env直接使用禁止](#processenv直接使用禁止)
- [設定構造](#設定構造)
- [環境別設定](#環境別設定)
- [シークレット管理](#シークレット管理)
- [マイグレーションガイド](#マイグレーションガイド)

---

## UnifiedConfig概要

### UnifiedConfigを使用する理由

**process.envの問題点:**
- ❌ タイプセーフティなし
- ❌ バリデーションなし
- ❌ テストしにくい
- ❌ コード全体に分散
- ❌ デフォルト値なし
- ❌ タイポに対するランタイムエラー

**unifiedConfigの利点:**
- ✅ タイプセーフな設定
- ✅ 単一の信頼できるソース
- ✅ 起動時のバリデーション
- ✅ モックで簡単にテスト
- ✅ 明確な構造
- ✅ 環境変数へのフォールバック

---

## process.env直接使用禁止

### ルール

```typescript
// ❌ 絶対にこうしないでください
const timeout = parseInt(process.env.TIMEOUT_MS || '5000');
const dbHost = process.env.DB_HOST || 'localhost';

// ✅ 常にこうしてください
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
const dbHost = config.database.host;
```

### これが重要な理由

**問題例:**
```typescript
// 環境変数名のタイポ
const host = process.env.DB_HSOT; // undefined！エラーなし！

// タイプセーフティ
const port = process.env.PORT; // 文字列！parseInt必要
const timeout = parseInt(process.env.TIMEOUT); // 設定なしだとNaN！
```

**unifiedConfig使用時:**
```typescript
const port = config.server.port; // number、保証される
const timeout = config.timeouts.default; // number、フォールバック含む
```

---

## 設定構造

### UnifiedConfigインターフェース

```typescript
export interface UnifiedConfig {
    database: {
        host: string;
        port: number;
        username: string;
        password: string;
        database: string;
    };
    server: {
        port: number;
        sessionSecret: string;
    };
    tokens: {
        jwt: string;
        inactivity: string;
        internal: string;
    };
    keycloak: {
        realm: string;
        client: string;
        baseUrl: string;
        secret: string;
    };
    aws: {
        region: string;
        emailQueueUrl: string;
        accessKeyId: string;
        secretAccessKey: string;
    };
    sentry: {
        dsn: string;
        environment: string;
        tracesSampleRate: number;
    };
    // ... その他のセクション
}
```

### 実装パターン

**ファイル:** `/blog-api/src/config/unifiedConfig.ts`

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as ini from 'ini';

const configPath = path.join(__dirname, '../../config.ini');
const iniConfig = ini.parse(fs.readFileSync(configPath, 'utf-8'));

export const config: UnifiedConfig = {
    database: {
        host: iniConfig.database?.host || process.env.DB_HOST || 'localhost',
        port: parseInt(iniConfig.database?.port || process.env.DB_PORT || '3306'),
        username: iniConfig.database?.username || process.env.DB_USER || 'root',
        password: iniConfig.database?.password || process.env.DB_PASSWORD || '',
        database: iniConfig.database?.database || process.env.DB_NAME || 'blog_dev',
    },
    server: {
        port: parseInt(iniConfig.server?.port || process.env.PORT || '3002'),
        sessionSecret: iniConfig.server?.sessionSecret || process.env.SESSION_SECRET || 'dev-secret',
    },
    // ... その他の設定
};

// 重要な設定のバリデーション
if (!config.tokens.jwt) {
    throw new Error('JWT secret not configured!');
}
```

**キーポイント:**
- config.iniから最初に読み取り
- process.envへフォールバック
- 開発用デフォルト値
- 起動時のバリデーション
- タイプセーフなアクセス

---

## 環境別設定

### config.ini構造

```ini
[database]
host = localhost
port = 3306
username = root
password = password1
database = blog_dev

[server]
port = 3002
sessionSecret = your-secret-here

[tokens]
jwt = your-jwt-secret
inactivity = 30m
internal = internal-api-token

[keycloak]
realm = myapp
client = myapp-client
baseUrl = http://localhost:8080
secret = keycloak-client-secret

[sentry]
dsn = https://your-sentry-dsn
environment = development
tracesSampleRate = 0.1
```

### 環境オーバーライド

```bash
# .envファイル（オプションのオーバーライド）
DB_HOST=production-db.example.com
DB_PASSWORD=secure-password
PORT=80
```

**優先順位:**
1. config.ini（最高優先）
2. process.env変数
3. ハードコードされたデフォルト値（最低優先）

---

## シークレット管理

### シークレットをコミットしない

```gitignore
# .gitignore
config.ini
.env
sentry.ini
*.pem
*.key
```

### 本番環境で環境変数使用

```typescript
// 開発: config.ini
// 本番: 環境変数

export const config: UnifiedConfig = {
    database: {
        password: process.env.DB_PASSWORD || iniConfig.database?.password || '',
    },
    tokens: {
        jwt: process.env.JWT_SECRET || iniConfig.tokens?.jwt || '',
    },
};
```

---

## マイグレーションガイド

### すべてのprocess.env使用を見つける

```bash
grep -r "process.env" blog-api/src/ --include="*.ts" | wc -l
```

### マイグレーション例

**以前:**
```typescript
// コード全体に分散
const timeout = parseInt(process.env.OPENID_HTTP_TIMEOUT_MS || '15000');
const keycloakUrl = process.env.KEYCLOAK_BASE_URL;
const jwtSecret = process.env.JWT_SECRET;
```

**以後:**
```typescript
import { config } from './config/unifiedConfig';

const timeout = config.keycloak.timeout;
const keycloakUrl = config.keycloak.baseUrl;
const jwtSecret = config.tokens.jwt;
```

**利点:**
- タイプセーフ
- 中央集権化
- テストしやすい
- 起動時のバリデーション

---

**関連ファイル:**
- [SKILL.md](SKILL.md)
- [testing-guide.md](testing-guide.md)
