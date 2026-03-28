# 🇨🇳 中文开发者 Agent Skills

[![Skills](https://img.shields.io/badge/agent--skills-compatible-blue)](https://skills.sh)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

聚焦开发者日常痛点的 Agent Skills。不是大而全的手册，每个 skill 只解决一个具体问题，但解决得很深。

适用于 Claude Code / Cursor / Kiro / Codex / OpenCode 等所有支持 Agent Skills 标准的工具。

## 安装

```bash
# 安装全部
npx skills add nanami7777777/chinese-fullstack-skills

# 安装单个
npx skills add nanami7777777/chinese-fullstack-skills/git-commit-cn
npx skills add nanami7777777/chinese-fullstack-skills/api-error-handling
npx skills add nanami7777777/chinese-fullstack-skills/code-review-cn
npx skills add nanami7777777/chinese-fullstack-skills/env-config-safety
```

## Skills

| Skill | 解决什么痛点 |
|-------|-------------|
| [git-commit-cn](./git-commit-cn/) | 写不好 commit message？中文 Conventional Commits 规范 + 从 diff 自动生成的决策逻辑 |
| [api-error-handling](./api-error-handling/) | 错误处理一团糟？HTTP 状态码决策树 + 数据库/第三方/并发等 6 种场景的完整处理方案 |
| [code-review-cn](./code-review-cn/) | Review 评论写不清楚？分级标注（🔴🟡🟢）+ 检查清单 + PR 描述模板 |
| [env-config-safety](./env-config-safety/) | 密钥泄露过？.env 规范 + .gitignore 模板 + 启动时验证 + 多环境密钥管理 |

## 设计理念

- **聚焦痛点**：每个 skill 只解决一个问题，不做大而全的手册
- **决策导向**：不只告诉你"怎么做"，更告诉你"什么时候该用哪种方案"
- **真实场景**：代码示例来自真实项目，包含边界情况和错误处理
- **中文优先**：scope 用中文业务域、错误信息用中文、注释用中文

## 贡献

欢迎 PR！好的 skill 应该：
- 只解决一个具体痛点
- 包含决策逻辑（什么时候用/什么时候不用）
- 代码示例包含边界情况
- 中文内容，description 用英文（agent 触发匹配用）

## License

MIT
