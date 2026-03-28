---
name: api-error-handling
description: Use this skill when implementing error handling in APIs, backend services, or full-stack applications. It provides decision trees for when to use which HTTP status code, how to structure error responses for Chinese teams, how to handle database errors, third-party API failures, validation errors, and authentication errors with proper logging and user-facing messages.
---

# API 错误处理决策指南

这个 skill 不是教你"什么是错误处理"，而是帮你在具体场景下做出正确的决策。

## 核心原则

1. **对外友好，对内详细** — 用户看到的是"操作失败，请稍后重试"，日志里记的是完整的错误栈和上下文
2. **错误要可追踪** — 每个错误响应带一个 traceId，用户反馈时能快速定位
3. **不要吞掉错误** — `catch (e) {}` 是最大的罪恶
4. **区分可恢复和不可恢复** — 网络超时可以重试，数据不一致不能

## HTTP 状态码决策树

遇到错误时，按这个顺序判断用哪个状态码：

```
请求有问题吗？
├── 参数格式错误 → 400 Bad Request
├── 没有登录 → 401 Unauthorized
├── 登录了但没权限 → 403 Forbidden
├── 资源不存在 → 404 Not Found
├── 资源已存在（如邮箱重复） → 409 Conflict
├── 请求体太大 → 413 Payload Too Large
├── 请求太频繁 → 429 Too Many Requests
└── 参数合法但业务规则不允许 → 422 Unprocessable Entity
    例：余额不足、库存不够、已过截止时间

服务端有问题吗？
├── 代码 bug → 500 Internal Server Error
├── 依赖服务挂了 → 502 Bad Gateway
├── 服务过载 → 503 Service Unavailable
└── 依赖服务超时 → 504 Gateway Timeout
```

### 最常犯的错误

```
✗ 所有错误都返回 200 + { code: -1 }     ← 前端无法用 HTTP 拦截器统一处理
✗ 所有错误都返回 500                     ← 客户端无法区分是自己的问题还是服务端的问题
✗ 404 和 403 混用                       ← 安全问题：403 泄露了资源存在的信息
✗ 业务错误用 400                        ← 400 是"请求格式错误"，不是"业务规则不允许"
```

## 错误响应格式

```typescript
// 统一错误响应结构
interface ErrorResponse {
  code: number          // 业务错误码（不是 HTTP 状态码）
  message: string       // 用户可见的错误信息（中文）
  traceId: string       // 追踪 ID，用于日志关联
  errors?: FieldError[] // 字段级错误（仅参数验证时）
}

interface FieldError {
  field: string         // 字段名
  message: string       // 该字段的错误信息
}
```

### 业务错误码设计

```
错误码 = HTTP状态码前缀 + 模块编号 + 序号

模块编号：
  01 = 用户
  02 = 订单
  03 = 支付
  04 = 商品
  05 = 通知

示例：
  400101 = 用户模块参数错误（用户名格式不对）
  401011 = 用户模块未登录
  403012 = 用户模块无权限
  404021 = 订单模块订单不存在
  409011 = 用户模块邮箱已注册
  422031 = 支付模块余额不足
```

## 具体场景处理

### 场景 1：数据库错误

```typescript
async function createUser(data: CreateUserInput) {
  try {
    return await prisma.user.create({ data })
  } catch (error) {
    // 唯一约束冲突 → 409
    if (error.code === 'P2002') {
      const field = error.meta?.target?.[0] ?? 'unknown'
      throw new AppError(409, `${field} 已存在`, { field })
    }

    // 外键约束失败 → 422
    if (error.code === 'P2003') {
      throw new AppError(422, '关联数据不存在')
    }

    // 连接失败 → 503（不是 500，因为是依赖问题不是代码 bug）
    if (error.code === 'P1001') {
      logger.error('数据库连接失败', { error })
      throw new AppError(503, '服务暂时不可用，请稍后重试')
    }

    // 其他未知数据库错误 → 500
    logger.error('未知数据库错误', { error, input: data })
    throw new AppError(500, '操作失败，请稍后重试')
  }
}
```

### 场景 2：第三方 API 调用失败

```typescript
async function sendSMS(phone: string, code: string) {
  try {
    const result = await smsProvider.send(phone, code)
    if (!result.success) {
      // 第三方返回业务错误（如手机号格式不对）→ 透传给用户
      throw new AppError(422, result.message)
    }
    return result
  } catch (error) {
    if (error instanceof AppError) throw error

    // 网络超时 → 可重试
    if (error.code === 'ETIMEDOUT' || error.code === 'ECONNREFUSED') {
      logger.warn('短信服务超时，准备重试', { phone, error: error.message })
      // 重试一次
      try {
        return await smsProvider.send(phone, code)
      } catch (retryError) {
        logger.error('短信服务重试失败', { phone, error: retryError })
        throw new AppError(503, '短信发送失败，请稍后重试')
      }
    }

    // 其他错误
    logger.error('短信发送未知错误', { phone, error })
    throw new AppError(500, '短信发送失败')
  }
}
```

### 场景 3：参数验证错误

```typescript
// 用 Zod 验证，统一转换为 400 + 字段级错误
function validateRequest<T>(schema: ZodSchema<T>, data: unknown): T {
  const result = schema.safeParse(data)
  if (!result.success) {
    const errors = result.error.issues.map(issue => ({
      field: issue.path.join('.'),
      message: issue.message,
    }))
    throw new ValidationError(errors)
  }
  return result.data
}

// 中间件统一处理
function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  const traceId = req.headers['x-trace-id'] as string ?? generateTraceId()

  if (err instanceof ValidationError) {
    return res.status(400).json({
      code: 400000,
      message: '参数验证失败',
      traceId,
      errors: err.errors,
    })
  }

  if (err instanceof AppError) {
    return res.status(err.httpStatus).json({
      code: err.code,
      message: err.message,
      traceId,
    })
  }

  // 未知错误 — 记录完整信息，但不暴露给用户
  logger.error('未处理的错误', {
    traceId,
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    body: req.body,
    userId: req.user?.id,
  })

  res.status(500).json({
    code: 500000,
    message: '服务器内部错误，请稍后重试',
    traceId,
  })
}
```

### 场景 4：并发冲突

```typescript
// 乐观锁处理并发更新
async function updateOrderStatus(orderId: string, newStatus: string, version: number) {
  const result = await prisma.order.updateMany({
    where: { id: orderId, version },
    data: { status: newStatus, version: version + 1 },
  })

  if (result.count === 0) {
    // 版本号不匹配 → 被其他请求先更新了
    throw new AppError(409, '订单状态已变更，请刷新后重试')
  }
}
```

## 前端错误处理配合

```typescript
// axios 拦截器 — 统一处理后端错误
axios.interceptors.response.use(
  (response) => response.data,
  (error) => {
    const status = error.response?.status
    const data = error.response?.data

    switch (status) {
      case 400:
        // 参数错误 — 显示字段级错误
        if (data.errors?.length) {
          // 交给表单组件显示
          return Promise.reject({ type: 'validation', errors: data.errors })
        }
        message.error(data.message ?? '请求参数错误')
        break

      case 401:
        // 未登录 — 跳转登录页
        userStore.logout()
        router.push('/login')
        break

      case 403:
        message.error('没有权限执行此操作')
        break

      case 409:
        // 冲突 — 提示用户刷新
        message.warning(data.message ?? '数据已变更，请刷新页面')
        break

      case 422:
        // 业务规则不允许 — 直接显示后端消息
        message.error(data.message)
        break

      case 429:
        message.warning('操作太频繁，请稍后再试')
        break

      case 500:
      case 502:
      case 503:
        message.error('服务暂时不可用，请稍后重试')
        // 可以上报 traceId 方便排查
        if (data.traceId) {
          console.error(`错误追踪 ID: ${data.traceId}`)
        }
        break

      default:
        message.error('网络请求失败')
    }

    return Promise.reject(error)
  }
)
```

## 日志规范

```typescript
// 错误日志必须包含的信息
logger.error('描述发生了什么', {
  traceId,           // 追踪 ID
  userId,            // 谁触发的
  action: '创建订单', // 在做什么操作
  input: { ... },    // 输入参数（脱敏后）
  error: err.message,// 错误信息
  stack: err.stack,  // 错误栈（仅 500 错误）
})

// 脱敏规则
// 手机号：138****1234
// 身份证：110***********1234
// 银行卡：6222 **** **** 1234
// 密码：永远不记录
```
