---
name: aliyun-deploy
description: Use this skill when deploying applications to Alibaba Cloud (Aliyun). It provides Chinese-language best practices for ECS deployment, OSS static hosting, CDN configuration, RDS database setup, Docker containerization, Nginx configuration, SSL certificates, domain ICP filing, and CI/CD with GitHub Actions for Chinese cloud infrastructure.
---

# 阿里云部署最佳实践

## 常见架构

```
用户 → CDN（阿里云 CDN）→ SLB（负载均衡）→ ECS（应用服务器）→ RDS（数据库）
                                                    ↓
                                              Redis（缓存）
                                                    ↓
                                              OSS（文件存储）
```

### 前后端分离部署
- 前端：OSS + CDN（静态资源托管）
- 后端：ECS + Docker（API 服务）
- 数据库：RDS MySQL / PostgreSQL
- 缓存：Redis
- 文件：OSS

## 前端部署（OSS + CDN）

### 1. 创建 OSS Bucket
```bash
# 安装 ossutil
brew install aliyun-cli

# 创建 Bucket（华东2-上海）
aliyun oss mb oss://my-frontend-bucket --region cn-shanghai

# 设置静态网站托管
aliyun oss website put oss://my-frontend-bucket \
  --index index.html \
  --error index.html  # SPA 路由回退
```

### 2. 上传构建产物
```bash
# 构建前端项目
pnpm build

# 上传到 OSS
aliyun oss sync dist/ oss://my-frontend-bucket/ \
  --delete \
  --include "*.html" --meta "Cache-Control:no-cache" \
  --exclude "*.html" --meta "Cache-Control:max-age=31536000"
```

### 3. 配置 CDN
- 加速域名：`www.example.com`
- 源站：OSS Bucket 域名
- 回源 HOST：Bucket 域名
- HTTPS：上传 SSL 证书或使用免费证书
- 缓存规则：HTML 不缓存，JS/CSS/图片长缓存

### GitHub Actions 自动部署
```yaml
# .github/workflows/deploy-frontend.yml
name: Deploy Frontend

on:
  push:
    branches: [main]
    paths: ['frontend/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with: { version: 9 }

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
          cache-dependency-path: frontend/pnpm-lock.yaml

      - run: pnpm install --frozen-lockfile
        working-directory: frontend

      - run: pnpm build
        working-directory: frontend

      - uses: manyuanrong/setup-ossutil@v3.0
        with:
          access-key-id: ${{ secrets.ALIYUN_ACCESS_KEY_ID }}
          access-key-secret: ${{ secrets.ALIYUN_ACCESS_KEY_SECRET }}
          endpoint: oss-cn-shanghai.aliyuncs.com

      - run: ossutil sync frontend/dist/ oss://my-frontend-bucket/ --delete -f
```

## 后端部署（ECS + Docker）

### Dockerfile
```dockerfile
# 多阶段构建 — Node.js 示例
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```dockerfile
# Go 示例
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server cmd/server/main.go

FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
ENV TZ=Asia/Shanghai
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

### Docker Compose（生产环境）
```yaml
# docker-compose.prod.yml
services:
  app:
    image: registry.cn-shanghai.aliyuncs.com/myns/myapp:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    restart: always
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: always
```

### Nginx 配置
```nginx
# nginx/conf.d/api.conf
server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # 安全头
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    location / {
        proxy_pass http://app:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # 静态文件缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}
```

## RDS 数据库

### 连接配置
```bash
# 环境变量
DATABASE_URL=mysql://user:password@rm-xxx.mysql.rds.aliyuncs.com:3306/mydb?ssl=true

# PostgreSQL
DATABASE_URL=postgresql://user:password@pgm-xxx.pg.rds.aliyuncs.com:5432/mydb?sslmode=require
```

### 安全规范
- 开启 SSL 连接
- 设置 IP 白名单（只允许 ECS 内网 IP）
- 定期备份（RDS 自动备份 + 手动备份）
- 生产环境用高可用版（主备切换）
- 不要用 root 账号连接应用

## 域名与备案

### ICP 备案流程
1. 在阿里云备案系统提交资料
2. 等待管局审核（约 5-20 个工作日）
3. 备案通过后在网站底部添加备案号
4. 备案号格式：`京ICP备XXXXXXXX号`

### DNS 配置
```
# A 记录 — 指向 ECS 公网 IP
api.example.com  →  A  →  47.xxx.xxx.xxx

# CNAME — 指向 CDN 域名
www.example.com  →  CNAME  →  xxx.cdn.aliyuncs.com

# MX 记录 — 企业邮箱
example.com  →  MX  →  mx1.mxhichina.com (优先级 10)
```

## 监控与告警

- 云监控：CPU、内存、磁盘、网络
- 应用监控：ARMS（应用实时监控服务）
- 日志：SLS（日志服务）收集 Docker 日志
- 告警规则：CPU > 80%、磁盘 > 90%、5xx 错误率 > 1%

## 成本优化

- 开发/测试环境用按量付费，生产用包年包月
- 静态资源走 CDN，减少 ECS 带宽消耗
- RDS 选择合适规格，不要过度配置
- OSS 设置生命周期规则，自动转低频/归档存储
- 使用竞价实例跑批处理任务
