---
name: env-config-safety
description: Use this skill when working with environment variables, configuration files, API keys, secrets, or .env files. It provides rules for secret management, .env file patterns, .gitignore configuration, and prevents accidental exposure of credentials in code, logs, or git history. Triggers when creating .env files, configuring API keys, setting up deployment, or handling sensitive data.
---

# 环境变量与密钥安全

## 绝对不能做的事

1. **永远不要把密钥提交到 Git**
   - 不管是 `.env`、`config.json`、还是硬编码在代码里
   - 一旦提交，即使删除了，Git 历史里还在
   - 如果已经提交了：立即轮换密钥，然后用 `git filter-branch` 或 BFG 清理历史

2. **永远不要在日志里打印密钥**
   ```typescript
   // ✗ 绝对不要
   console.log('Config:', config)  // config 里可能有 apiKey
   logger.info('Request headers:', req.headers)  // headers 里有 Authorization

   // ✓ 只打印需要的字段
   logger.info('Config loaded', { model: config.model, contextLimit: config.contextLimit })
   ```

3. **永远不要在前端代码里放密钥**
   - 前端代码会被用户看到，无论你怎么混淆
   - 需要调用第三方 API？通过后端代理

## .gitignore 必须包含的内容

```gitignore
# 环境变量文件
.env
.env.local
.env.*.local
.env.development
.env.production

# 但保留示例文件
!.env.example

# IDE 配置（可能包含路径信息）
.vscode/settings.json
.idea/

# 操作系统文件
.DS_Store
Thumbs.db

# 依赖
node_modules/

# 构建产物
dist/
build/
.next/
.nuxt/

# 日志
*.log
logs/

# 密钥文件
*.pem
*.key
*.p12
*.pfx
```

## .env 文件规范

### .env.example（提交到 Git，给团队参考）

```bash
# .env.example — 所有需要的环境变量，值用占位符

# 应用
NODE_ENV=development
PORT=3000

# 数据库
DATABASE_URL=postgresql://user:password@localhost:5432/mydb

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-jwt-secret-at-least-32-chars
JWT_EXPIRES_IN=7d

# 第三方服务
ALIYUN_ACCESS_KEY_ID=your-access-key-id
ALIYUN_ACCESS_KEY_SECRET=your-access-key-secret
ALIYUN_OSS_BUCKET=your-bucket-name
ALIYUN_OSS_REGION=oss-cn-shanghai

# 微信
WECHAT_APP_ID=your-app-id
WECHAT_APP_SECRET=your-app-secret

# 短信
SMS_ACCESS_KEY=your-sms-key
SMS_SIGN_NAME=你的签名
SMS_TEMPLATE_CODE=SMS_123456
```

### 规则
- `.env.example` 提交到 Git，`.env` 不提交
- 每个环境变量都要有注释说明用途
- 密钥类的值用 `your-xxx` 占位符
- 新增环境变量时必须同步更新 `.env.example`

## 代码中读取环境变量的方式

### 不要到处 process.env

```typescript
// ✗ 散落在各处
const token = jwt.sign(payload, process.env.JWT_SECRET!)  // 如果忘了设置就是 undefined

// ✓ 集中管理 + 启动时验证
// config/env.ts
import { z } from 'zod'

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET 至少 32 个字符'),
  JWT_EXPIRES_IN: z.string().default('7d'),
  REDIS_URL: z.string().url().optional(),
})

// 启动时验证，缺少必要变量直接报错退出
function loadEnv() {
  const result = envSchema.safeParse(process.env)
  if (!result.success) {
    console.error('❌ 环境变量配置错误:')
    for (const issue of result.error.issues) {
      console.error(`   ${issue.path.join('.')}: ${issue.message}`)
    }
    process.exit(1)
  }
  return result.data
}

export const env = loadEnv()

// 使用时
import { env } from '@/config/env'
const token = jwt.sign(payload, env.JWT_SECRET)  // 类型安全，保证有值
```

## 不同环境的密钥管理

| 环境 | 密钥存放位置 | 说明 |
|------|-------------|------|
| 本地开发 | `.env` 文件 | 每个开发者自己的文件，不提交 |
| CI/CD | GitHub Secrets / GitLab Variables | 在 CI 平台的设置里配置 |
| 生产环境 | 云平台密钥管理服务 | 阿里云 KMS / AWS Secrets Manager |
| Docker | `docker-compose.yml` 的 `env_file` | 或者 Docker Secrets |

### GitHub Actions 中使用密钥

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          JWT_SECRET: ${{ secrets.JWT_SECRET }}
```

### Docker Compose 中使用密钥

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    env_file:
      - .env.production  # 这个文件在服务器上，不在 Git 里
```

## 密钥轮换

- 定期轮换密钥（至少每 90 天）
- 轮换时先添加新密钥，确认新密钥生效后再删除旧密钥
- JWT_SECRET 轮换时要考虑已签发的 token 失效问题
- 数据库密码轮换时要确保所有服务都更新了

## 前端环境变量

```bash
# Vite 项目：只有 VITE_ 前缀的变量会暴露给前端
VITE_API_BASE_URL=https://api.example.com
VITE_APP_TITLE=我的应用

# Next.js 项目：只有 NEXT_PUBLIC_ 前缀的变量会暴露给前端
NEXT_PUBLIC_API_URL=https://api.example.com

# ⚠️ 这些变量会被打包到前端代码里，用户可以看到
# 所以绝对不要放密钥！
```
