---
name: node-api-design
description: Use this skill when building Node.js backend APIs. It provides Chinese-language best practices for RESTful API design with Express or Fastify, TypeScript, database access with Prisma/Drizzle, authentication with JWT, error handling, validation with Zod, and project structure conventions for Chinese development teams.
---

# Node.js RESTful API 中文开发规范

## 项目结构

```
src/
├── controllers/        # 控制器：处理请求和响应
├── services/           # 服务层：业务逻辑
├── models/             # 数据模型（Prisma schema 或 Drizzle schema）
├── middlewares/         # 中间件（认证、日志、错误处理）
├── routes/             # 路由定义
├── validators/         # 请求参数验证（Zod schema）
├── utils/              # 工具函数
├── types/              # TypeScript 类型定义
├── config/             # 配置文件
│   ├── index.ts        # 统一配置入口
│   └── database.ts     # 数据库配置
├── app.ts              # Express/Fastify 实例
└── server.ts           # 启动入口
```

## 分层架构

```
请求 → 路由 → 中间件 → 控制器 → 服务层 → 数据访问层 → 数据库
                                    ↓
                              响应 ← 控制器
```

### 控制器（Controller）
只负责：接收请求参数、调用服务层、返回响应。不写业务逻辑。

```typescript
// controllers/userController.ts
import { Request, Response, NextFunction } from 'express'
import { userService } from '@/services/userService'
import { CreateUserSchema, UserQuerySchema } from '@/validators/userValidator'

export const userController = {
  /** 获取用户列表 */
  async getList(req: Request, res: Response, next: NextFunction) {
    try {
      const query = UserQuerySchema.parse(req.query)
      const result = await userService.getList(query)
      res.json({ code: 0, data: result, message: '成功' })
    } catch (error) {
      next(error)
    }
  },

  /** 创建用户 */
  async create(req: Request, res: Response, next: NextFunction) {
    try {
      const data = CreateUserSchema.parse(req.body)
      const user = await userService.create(data)
      res.status(201).json({ code: 0, data: user, message: '创建成功' })
    } catch (error) {
      next(error)
    }
  },
}
```

### 服务层（Service）
负责业务逻辑，不依赖 Express 的 req/res。

```typescript
// services/userService.ts
import { prisma } from '@/config/database'
import { AppError } from '@/utils/errors'

export const userService = {
  async getList(query: UserQuery) {
    const { page = 1, pageSize = 20, keyword } = query
    const where = keyword
      ? { name: { contains: keyword } }
      : {}

    const [list, total] = await Promise.all([
      prisma.user.findMany({
        where,
        skip: (page - 1) * pageSize,
        take: pageSize,
        orderBy: { createdAt: 'desc' },
      }),
      prisma.user.count({ where }),
    ])

    return { list, total, page, pageSize }
  },

  async create(data: CreateUserInput) {
    // 检查邮箱是否已存在
    const existing = await prisma.user.findUnique({
      where: { email: data.email },
    })
    if (existing) {
      throw new AppError(409, '该邮箱已被注册')
    }

    return prisma.user.create({ data })
  },
}
```

## 统一响应格式

```typescript
// 成功响应
{
  "code": 0,
  "data": { ... },
  "message": "成功"
}

// 列表响应
{
  "code": 0,
  "data": {
    "list": [...],
    "total": 100,
    "page": 1,
    "pageSize": 20
  },
  "message": "成功"
}

// 错误响应
{
  "code": 40001,
  "message": "参数验证失败",
  "errors": [
    { "field": "email", "message": "邮箱格式不正确" }
  ]
}
```

## 错误处理

```typescript
// utils/errors.ts
export class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public code?: number,
  ) {
    super(message)
    this.name = 'AppError'
  }
}

// middlewares/errorHandler.ts
import { Request, Response, NextFunction } from 'express'
import { ZodError } from 'zod'
import { AppError } from '@/utils/errors'

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction,
) {
  // Zod 验证错误
  if (err instanceof ZodError) {
    return res.status(400).json({
      code: 40001,
      message: '参数验证失败',
      errors: err.errors.map(e => ({
        field: e.path.join('.'),
        message: e.message,
      })),
    })
  }

  // 业务错误
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      code: err.code ?? err.statusCode,
      message: err.message,
    })
  }

  // 未知错误
  console.error('未处理的错误:', err)
  res.status(500).json({
    code: 50000,
    message: '服务器内部错误',
  })
}
```

## 参数验证（Zod）

```typescript
// validators/userValidator.ts
import { z } from 'zod'

export const CreateUserSchema = z.object({
  name: z.string().min(2, '姓名至少 2 个字符').max(50),
  email: z.string().email('邮箱格式不正确'),
  password: z.string().min(8, '密码至少 8 位'),
  role: z.enum(['admin', 'editor', 'viewer']).default('viewer'),
})

export const UserQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
  keyword: z.string().optional(),
  status: z.enum(['active', 'inactive']).optional(),
})

export type CreateUserInput = z.infer<typeof CreateUserSchema>
export type UserQuery = z.infer<typeof UserQuerySchema>
```

## JWT 认证

```typescript
// middlewares/auth.ts
import jwt from 'jsonwebtoken'
import { AppError } from '@/utils/errors'

const JWT_SECRET = process.env.JWT_SECRET!

export function authMiddleware(req: Request, _res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '')

  if (!token) {
    throw new AppError(401, '请先登录')
  }

  try {
    const payload = jwt.verify(token, JWT_SECRET) as JwtPayload
    req.user = payload
    next()
  } catch {
    throw new AppError(401, '登录已过期，请重新登录')
  }
}
```

## 数据库规范

- 表名用复数小写：`users`、`orders`、`order_items`
- 字段名用 snake_case：`created_at`、`updated_at`
- 每张表必须有 `id`、`created_at`、`updated_at`
- 软删除用 `deleted_at` 字段
- 索引命名：`idx_表名_字段名`
- 外键命名：`fk_表名_关联表名`

## 环境变量

```bash
# .env.example
NODE_ENV=development
PORT=3000

# 数据库
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=7d

# Redis（可选）
REDIS_URL=redis://localhost:6379
```

- 敏感信息不要提交到 Git
- 用 `.env.example` 记录所有需要的环境变量
- 不同环境用 `.env.development`、`.env.production`
