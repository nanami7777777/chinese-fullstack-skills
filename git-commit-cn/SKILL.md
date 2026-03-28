---
name: git-commit-cn
description: Use this skill when writing git commit messages or generating changelogs. It enforces Conventional Commits format with Chinese scope descriptions, provides templates for common commit types, and generates structured commit messages from code diffs. Triggers on git commit, changelog generation, or version release tasks.
---

# Git 提交信息规范

## 核心规则

每次提交必须遵循 Conventional Commits 格式：

```
<type>(<scope>): <subject>

<body>

<footer>
```

### type 必须是以下之一

| type | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(用户): 添加手机号登录` |
| `fix` | 修复 bug | `fix(订单): 修复金额计算精度丢失` |
| `docs` | 文档变更 | `docs: 更新 API 接口文档` |
| `style` | 代码格式（不影响逻辑） | `style: 统一缩进为 2 空格` |
| `refactor` | 重构（不是新功能也不是修 bug） | `refactor(支付): 抽取公共支付逻辑` |
| `perf` | 性能优化 | `perf(列表): 虚拟滚动优化长列表渲染` |
| `test` | 测试相关 | `test(用户): 补充注册接口单元测试` |
| `chore` | 构建/工具/依赖变更 | `chore: 升级 vite 到 5.x` |
| `ci` | CI/CD 配置 | `ci: 添加 GitHub Actions 自动部署` |
| `revert` | 回滚 | `revert: 回滚 feat(用户): 添加手机号登录` |

### scope 用中文业务域

scope 描述影响范围，用中文业务域名称，不用文件名：

```
✓ feat(用户): 添加头像上传功能
✓ fix(订单): 修复取消订单后库存未恢复
✓ feat(权限): 支持按钮级别权限控制
✗ feat(src/views/user): 添加头像上传功能    ← 不要用文件路径
✗ feat(UserAvatar.vue): 添加头像上传功能    ← 不要用文件名
```

常见 scope：`用户`、`订单`、`支付`、`商品`、`权限`、`通知`、`搜索`、`报表`、`设置`

无明确业务域时可省略 scope：`docs: 更新部署文档`

### subject 规则

- 用中文，不超过 50 个字符
- 不加句号
- 用祈使语气："添加"而非"添加了"
- 说清楚做了什么，不要写"修改代码"、"更新文件"这种废话

```
✓ feat(购物车): 支持批量删除商品
✓ fix(登录): 修复验证码倒计时结束后按钮未恢复
✗ fix(登录): 修复了一个 bug          ← 没说修了什么
✗ feat: 更新代码                     ← 废话
✗ fix: 改了点东西                    ← 废话
```

## body 什么时候写

以下情况必须写 body：

1. **破坏性变更** — 说明什么变了、为什么变、怎么迁移
2. **复杂修复** — 说明根因是什么、为什么这样修
3. **重要决策** — 说明为什么选择这个方案而不是其他方案

```
fix(订单): 修复并发下单时库存扣减不一致

根因：多个请求同时读取库存后各自扣减，导致超卖。
方案：改用数据库乐观锁（version 字段），扣减时检查版本号。
放弃方案：Redis 分布式锁（引入额外依赖，且单点故障风险）。

影响范围：order_service.go 的 CreateOrder 方法。
```

## footer 什么时候写

```
# 关联 issue
fix(支付): 修复微信支付回调签名验证失败

Closes #142

# 破坏性变更
feat(API): 用户列表接口返回格式变更

BREAKING CHANGE: GET /api/users 返回格式从数组改为分页对象
迁移方式：将 response.data 改为 response.data.list
```

## 从 diff 生成 commit message 的决策逻辑

当需要根据代码变更生成 commit message 时，按以下步骤判断：

1. **判断 type**：
   - 新增了用户可感知的功能 → `feat`
   - 修复了已有功能的问题 → `fix`
   - 只改了代码结构没改行为 → `refactor`
   - 只改了测试 → `test`
   - 只改了文档 → `docs`
   - 只改了构建/依赖 → `chore`

2. **判断 scope**：
   - 看改动涉及哪个业务模块
   - 如果跨多个模块，用最主要的那个
   - 如果是基础设施变更，省略 scope

3. **写 subject**：
   - 从用户/调用方的角度描述变更效果
   - 不要描述实现细节（那是 body 的事）

4. **判断是否需要 body**：
   - 改动超过 100 行 → 写
   - 修复了一个不明显的 bug → 写根因
   - 有多种方案可选 → 写为什么选这个

## 不要做的事

- 不要一个 commit 里混多种不相关的改动
- 不要把 `console.log` 调试代码提交上去
- 不要提交 `.env` 文件或任何密钥
- 不要用 `git add .` 然后盲目提交，先 `git diff --staged` 看一眼
- 不要写英文 commit message 然后用翻译工具翻成中文（语感不对）
