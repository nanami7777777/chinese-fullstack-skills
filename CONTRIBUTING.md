# 贡献指南

欢迎为中文全栈开发 Skills 贡献内容！

## 如何贡献

1. Fork 本仓库
2. 创建新分支：`git checkout -b feat/my-new-skill`
3. 添加或修改 skill
4. 提交 PR

## Skill 格式要求

每个 skill 是一个独立目录，包含一个 `SKILL.md` 文件：

```
my-skill/
└── SKILL.md
```

### SKILL.md 格式

```markdown
---
name: my-skill-name
description: Use this skill when... (英文描述，用于 agent 自动匹配触发)
---

# Skill 标题（中文）

正文内容（中文）...
```

### 注意事项

- `name` 用 kebab-case
- `description` 用英文（因为 agent 用英文匹配触发条件）
- 正文内容用中文
- 代码示例要完整可运行
- 避免过于主观的技术选型建议，给出多个选项让用户选择
- 保持实用性，不要写空洞的理论

## 建议的 Skill 方向

- 更多框架：NestJS、Nuxt 3、Flutter
- 更多云平台：腾讯云、华为云
- 更多场景：微信公众号、支付宝小程序、抖音小程序
- 工具链：Vite 配置、ESLint/Prettier 规范、Monorepo
