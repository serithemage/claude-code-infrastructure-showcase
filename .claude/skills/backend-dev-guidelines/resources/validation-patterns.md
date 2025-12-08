# バリデーションパターン - Zodを使用した入力検証

タイプセーフなバリデーションのためのZodスキーマ使用に関する完全なガイドです。

## 目次

- [Zodを使用する理由](#zodを使用する理由)
- [基本Zodパターン](#基本zodパターン)
- [コードベーススキーマ例](#コードベーススキーマ例)
- [Routeレベルバリデーション](#routeレベルバリデーション)
- [Controllerバリデーション](#controllerバリデーション)
- [DTOパターン](#dtoパターン)
- [エラー処理](#エラー処理)
- [高度なパターン](#高度なパターン)

---

## Zodを使用する理由

### Joi/他のライブラリに対する利点

**タイプセーフティ:**
- ✅ 完全なTypeScript推論
- ✅ ランタイム + コンパイルタイムバリデーション
- ✅ 自動型生成

**開発者体験:**
- ✅ 直感的なAPI
- ✅ 組み合わせ可能なスキーマ
- ✅ 優れたエラーメッセージ

**パフォーマンス:**
- ✅ 高速なバリデーション
- ✅ 小さなバンドルサイズ
- ✅ Tree-shakeable

### Joiからのマイグレーション

モダンなバリデーションはJoiの代わりにZodを使用します:

```typescript
// ❌ 古い - Joi（段階的に廃止中）
const schema = Joi.object({
    email: Joi.string().email().required(),
    name: Joi.string().min(3).required(),
});

// ✅ 新しい - Zod（推奨）
const schema = z.object({
    email: z.string().email(),
    name: z.string().min(3),
});
```

---

## 基本Zodパターン

### 基本型

```typescript
import { z } from 'zod';

// 文字列
const nameSchema = z.string();
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();
const minLengthSchema = z.string().min(3);
const maxLengthSchema = z.string().max(100);

// 数値
const ageSchema = z.number().int().positive();
const priceSchema = z.number().positive();
const rangeSchema = z.number().min(0).max(100);

// ブール値
const activeSchema = z.boolean();

// 日付
const dateSchema = z.string().datetime(); // ISO 8601文字列
const nativeDateSchema = z.date(); // ネイティブDateオブジェクト

// 列挙型
const roleSchema = z.enum(['admin', 'operations', 'user']);
const statusSchema = z.enum(['PENDING', 'APPROVED', 'REJECTED']);
```

### オブジェクト

```typescript
// シンプルなオブジェクト
const userSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// ネストされたオブジェクト
const addressSchema = z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().regex(/^\d{5}$/),
});

const userWithAddressSchema = z.object({
    name: z.string(),
    address: addressSchema,
});

// オプショナルフィールド
const userSchema = z.object({
    name: z.string(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
});

// Nullableフィールド
const userSchema = z.object({
    name: z.string(),
    middleName: z.string().nullable(),
});
```

### 配列

```typescript
// 基本型の配列
const rolesSchema = z.array(z.string());
const numbersSchema = z.array(z.number());

// オブジェクトの配列
const usersSchema = z.array(
    z.object({
        id: z.string(),
        name: z.string(),
    })
);

// 制約のある配列
const tagsSchema = z.array(z.string()).min(1).max(10);
const nonEmptyArray = z.array(z.string()).nonempty();
```

---

## コードベーススキーマ例

### フォームバリデーションスキーマ

**ファイル:** `/form/src/helpers/zodSchemas.ts`

```typescript
import { z } from 'zod';

// 質問タイプ列挙型
export const questionTypeSchema = z.enum([
    'input',
    'textbox',
    'editor',
    'dropdown',
    'autocomplete',
    'checkbox',
    'radio',
    'upload',
]);

// アップロードタイプ
export const uploadTypeSchema = z.array(
    z.enum(['pdf', 'image', 'excel', 'video', 'powerpoint', 'word']).nullable()
);

// 入力タイプ
export const inputTypeSchema = z
    .enum(['date', 'number', 'input', 'currency'])
    .nullable();

// 質問オプション
export const questionOptionSchema = z.object({
    id: z.number().int().positive().optional(),
    controlTag: z.string().max(150).nullable().optional(),
    label: z.string().max(100).nullable().optional(),
    order: z.number().int().min(0).default(0),
});

// 質問スキーマ
export const questionSchema = z.object({
    id: z.number().int().positive().optional(),
    formID: z.number().int().positive(),
    sectionID: z.number().int().positive().optional(),
    options: z.array(questionOptionSchema).optional(),
    label: z.string().max(500),
    description: z.string().max(5000).optional(),
    type: questionTypeSchema,
    uploadTypes: uploadTypeSchema.optional(),
    inputType: inputTypeSchema.optional(),
    tags: z.array(z.string().max(150)).optional(),
    required: z.boolean(),
    isStandard: z.boolean().optional(),
    deprecatedKey: z.string().nullable().optional(),
    maxLength: z.number().int().positive().nullable().optional(),
    isOptionsSorted: z.boolean().optional(),
});

// フォームセクションスキーマ
export const formSectionSchema = z.object({
    id: z.number().int().positive(),
    formID: z.number().int().positive(),
    questions: z.array(questionSchema).optional(),
    label: z.string().max(500),
    description: z.string().max(5000).optional(),
    isStandard: z.boolean(),
});

// フォーム作成スキーマ
export const createFormSchema = z.object({
    id: z.number().int().positive(),
    label: z.string().max(150),
    description: z.string().max(6000).nullable().optional(),
    isPhase: z.boolean().optional(),
    username: z.string(),
});

// 順序更新スキーマ
export const updateOrderSchema = z.object({
    source: z.object({
        index: z.number().int().min(0),
        sectionID: z.number().int().min(0),
    }),
    destination: z.object({
        index: z.number().int().min(0),
        sectionID: z.number().int().min(0),
    }),
});

// Controller専用バリデーションスキーマ
export const createQuestionValidationSchema = z.object({
    formID: z.number().int().positive(),
    sectionID: z.number().int().positive(),
    question: questionSchema,
    index: z.number().int().min(0).nullable().optional(),
    username: z.string(),
});

export const updateQuestionValidationSchema = z.object({
    questionID: z.number().int().positive(),
    username: z.string(),
    question: questionSchema,
});
```

### プロキシ関係スキーマ

```typescript
// プロキシ関係バリデーション
const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

// カスタムバリデーション付き
const createProxySchemaWithValidation = createProxySchema.refine(
    (data) => new Date(data.expiresAt) > new Date(data.startsAt),
    {
        message: 'expiresAt must be after startsAt',
        path: ['expiresAt'],
    }
);
```

### Workflowバリデーション

```typescript
// Workflow開始スキーマ
const startWorkflowSchema = z.object({
    workflowCode: z.string().min(1),
    entityType: z.enum(['Post', 'User', 'Comment']),
    entityID: z.number().int().positive(),
    dryRun: z.boolean().optional().default(false),
});

// Workflowステップ完了スキーマ
const completeStepSchema = z.object({
    stepInstanceID: z.number().int().positive(),
    answers: z.record(z.string(), z.any()),
    dryRun: z.boolean().optional().default(false),
});
```

---

## Routeレベルバリデーション

### パターン1: インラインバリデーション

```typescript
// routes/proxyRoutes.ts
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

router.post(
    '/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => {
        try {
            // Routeレベルでバリデーション
            const validated = createProxySchema.parse(req.body);

            // Serviceに委譲
            const proxy = await proxyService.createProxyRelationship(validated);

            res.status(201).json({ success: true, data: proxy });
        } catch (error) {
            if (error instanceof z.ZodError) {
                return res.status(400).json({
                    success: false,
                    error: {
                        message: 'Validation failed',
                        details: error.errors,
                    },
                });
            }
            handler.handleException(res, error);
        }
    }
);
```

**利点:**
- 高速でシンプル
- シンプルなroutesに良い

**欠点:**
- バリデーションロジックがroutesに存在
- テストしにくい
- 再利用不可

---

## Controllerバリデーション

### パターン2: Controllerバリデーション（推奨）

```typescript
// validators/userSchemas.ts
import { z } from 'zod';

export const createUserSchema = z.object({
    email: z.string().email(),
    name: z.string().min(2).max(100),
    roles: z.array(z.enum(['admin', 'operations', 'user'])),
    isActive: z.boolean().default(true),
});

export const updateUserSchema = z.object({
    email: z.string().email().optional(),
    name: z.string().min(2).max(100).optional(),
    roles: z.array(z.enum(['admin', 'operations', 'user'])).optional(),
    isActive: z.boolean().optional(),
});

export type CreateUserDTO = z.infer<typeof createUserSchema>;
export type UpdateUserDTO = z.infer<typeof updateUserSchema>;
```

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema, updateUserSchema } from '../validators/userSchemas';
import { z } from 'zod';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // 入力バリデーション
            const validated = createUserSchema.parse(req.body);

            // Service呼び出し
            const user = await this.userService.createUser(validated);

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            if (error instanceof z.ZodError) {
                // 400ステータスでバリデーションエラー処理
                return this.handleError(error, res, 'createUser', 400);
            }
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            // paramsとbodyのバリデーション
            const userId = req.params.id;
            const validated = updateUserSchema.parse(req.body);

            const user = await this.userService.updateUser(userId, validated);

            this.handleSuccess(res, user, 'User updated successfully');
        } catch (error) {
            if (error instanceof z.ZodError) {
                return this.handleError(error, res, 'updateUser', 400);
            }
            this.handleError(error, res, 'updateUser');
        }
    }
}
```

**利点:**
- クリーンな分離
- 再利用可能なスキーマ
- テストしやすい
- タイプセーフなDTOs

**欠点:**
- 管理するファイルが多い

---

## DTOパターン

### スキーマから型を推論

```typescript
import { z } from 'zod';

// スキーマ定義
const createUserSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// スキーマからTypeScript型を推論
type CreateUserDTO = z.infer<typeof createUserSchema>;

// 以下と同等:
// type CreateUserDTO = {
//     email: string;
//     name: string;
//     age: number;
// }

// Serviceで使用
class UserService {
    async createUser(data: CreateUserDTO): Promise<User> {
        // dataは完全に型付けされている！
        console.log(data.email); // ✅ TypeScriptはこれが存在することを知っている
        console.log(data.invalid); // ❌ TypeScriptエラー！
    }
}
```

### 入力 vs 出力型

```typescript
// 入力スキーマ（APIが受け取るもの）
const createUserInputSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    password: z.string().min(8),
});

// 出力スキーマ（APIが返すもの）
const userOutputSchema = z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    name: z.string(),
    createdAt: z.string().datetime(),
    // passwordを除外！
});

type CreateUserInput = z.infer<typeof createUserInputSchema>;
type UserOutput = z.infer<typeof userOutputSchema>;
```

---

## エラー処理

### Zodエラーフォーマット

```typescript
try {
    const validated = schema.parse(data);
} catch (error) {
    if (error instanceof z.ZodError) {
        console.log(error.errors);
        // [
        //   {
        //     code: 'invalid_type',
        //     expected: 'string',
        //     received: 'number',
        //     path: ['email'],
        //     message: 'Expected string, received number'
        //   }
        // ]
    }
}
```

### カスタムエラーメッセージ

```typescript
const userSchema = z.object({
    email: z.string().email({ message: 'Please provide a valid email address' }),
    name: z.string().min(2, { message: 'Name must be at least 2 characters' }),
    age: z.number().int().positive({ message: 'Age must be a positive number' }),
});
```

### フォーマットされたエラーレスポンス

```typescript
// Zodエラーをフォーマットするヘルパー関数
function formatZodError(error: z.ZodError) {
    return {
        message: 'Validation failed',
        errors: error.errors.map((err) => ({
            field: err.path.join('.'),
            message: err.message,
            code: err.code,
        })),
    };
}

// Controllerで
catch (error) {
    if (error instanceof z.ZodError) {
        return res.status(400).json({
            success: false,
            error: formatZodError(error),
        });
    }
}

// レスポンス例:
// {
//   "success": false,
//   "error": {
//     "message": "Validation failed",
//     "errors": [
//       {
//         "field": "email",
//         "message": "Invalid email",
//         "code": "invalid_string"
//       }
//     ]
//   }
// }
```

---

## 高度なパターン

### 条件付きバリデーション

```typescript
// 他のフィールド値に基づくバリデーション
const submissionSchema = z.object({
    type: z.enum(['NEW', 'UPDATE']),
    postId: z.number().optional(),
}).refine(
    (data) => {
        // typeがUPDATEならpostIdは必須
        if (data.type === 'UPDATE') {
            return data.postId !== undefined;
        }
        return true;
    },
    {
        message: 'postId is required when type is UPDATE',
        path: ['postId'],
    }
);
```

### データ変換

```typescript
// 文字列を数値に変換
const userSchema = z.object({
    name: z.string(),
    age: z.string().transform((val) => parseInt(val, 10)),
});

// 日付変換
const eventSchema = z.object({
    name: z.string(),
    date: z.string().transform((str) => new Date(str)),
});
```

### データ前処理

```typescript
// バリデーション前に文字列をtrim
const userSchema = z.object({
    email: z.preprocess(
        (val) => typeof val === 'string' ? val.trim().toLowerCase() : val,
        z.string().email()
    ),
    name: z.preprocess(
        (val) => typeof val === 'string' ? val.trim() : val,
        z.string().min(2)
    ),
});
```

### ユニオン型

```typescript
// 複数の可能な型
const idSchema = z.union([z.string(), z.number()]);

// 判別可能なユニオン
const notificationSchema = z.discriminatedUnion('type', [
    z.object({
        type: z.literal('email'),
        recipient: z.string().email(),
        subject: z.string(),
    }),
    z.object({
        type: z.literal('sms'),
        phoneNumber: z.string(),
        message: z.string(),
    }),
]);
```

### 再帰スキーマ

```typescript
// ツリー状のネスト構造用
type Category = {
    id: number;
    name: string;
    children?: Category[];
};

const categorySchema: z.ZodType<Category> = z.lazy(() =>
    z.object({
        id: z.number(),
        name: z.string(),
        children: z.array(categorySchema).optional(),
    })
);
```

### スキーマ組み合わせ

```typescript
// 基本スキーマ
const timestampsSchema = z.object({
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime(),
});

const auditSchema = z.object({
    createdBy: z.string(),
    updatedBy: z.string(),
});

// スキーマの組み合わせ
const userSchema = z.object({
    id: z.string(),
    email: z.string().email(),
    name: z.string(),
}).merge(timestampsSchema).merge(auditSchema);

// スキーマの拡張
const adminUserSchema = userSchema.extend({
    adminLevel: z.number().int().min(1).max(5),
    permissions: z.array(z.string()),
});

// 特定フィールドの選択
const publicUserSchema = userSchema.pick({
    id: true,
    name: true,
    // emailを除外
});

// フィールドの除外
const userWithoutTimestamps = userSchema.omit({
    createdAt: true,
    updatedAt: true,
});
```

### Validation Middleware

```typescript
// 再利用可能なvalidation middlewareを作成
import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';

export function validateBody<T extends z.ZodType>(schema: T) {
    return (req: Request, res: Response, next: NextFunction) => {
        try {
            req.body = schema.parse(req.body);
            next();
        } catch (error) {
            if (error instanceof z.ZodError) {
                return res.status(400).json({
                    success: false,
                    error: {
                        message: 'Validation failed',
                        details: error.errors,
                    },
                });
            }
            next(error);
        }
    };
}

// 使用法
router.post('/users',
    validateBody(createUserSchema),
    async (req, res) => {
        // req.bodyがバリデートされて型付けされている！
        const user = await userService.createUser(req.body);
        res.json({ success: true, data: user });
    }
);
```

---

**関連ファイル:**
- [SKILL.md](SKILL.md) - メインガイド
- [routing-and-controllers.md](routing-and-controllers.md) - Controllerでバリデーションを使用
- [services-and-repositories.md](services-and-repositories.md) - ServicesでDTOsを使用
- [async-and-errors.md](async-and-errors.md) - エラー処理パターン
