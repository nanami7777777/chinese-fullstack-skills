# 🇨🇳 中文全栈开发 Agent Skills

[![Skills](https://img.shields.io/badge/agent--skills-compatible-blue)](https://skills.sh)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

面向中文开发者的 AI Agent Skills 合集。适用于 Claude Code / Cursor / Kiro / Codex / OpenCode 等所有支持 Agent Skills 标准的工具。

## 安装

```bash
# 安装全部 skills
npx skills add nanami7777777/chinese-fullstack-skills

# 安装单个 skill
npx skills add nanami7777777/chinese-fullstack-skills/vue3-best-practices
npx skills add nanami7777777/chinese-fullstack-skills/react-nextjs-cn
npx skills add nanami7777777/chinese-fullstack-skills/node-api-design
npx skills add nanami7777777/chinese-fullstack-skills/go-web-service
npx skills add nanami7777777/chinese-fullstack-skills/aliyun-deploy
npx skills add nanami7777777/chinese-fullstack-skills/wechat-miniprogram
```

## Skills 列表

| Skill | 说明 | 适用场景 |
|-------|------|----------|
| [vue3-best-practices](./vue3-best-practices/) | Vue 3 + TypeScript + Pinia 开发规范 | Vue 3 项目开发 |
| [react-nextjs-cn](./react-nextjs-cn/) | React 19 + Next.js 15 中文开发规范 | React/Next.js 项目 |
| [node-api-design](./node-api-design/) | Node.js RESTful API 设计规范 | 后端 API 开发 |
| [go-web-service](./go-web-service/) | Go Web 服务开发规范 | Go 后端服务 |
| [aliyun-deploy](./aliyun-deploy/) | 阿里云部署最佳实践 | 国内云部署 |
| [wechat-miniprogram](./wechat-miniprogram/) | 微信小程序开发规范 | 小程序开发 |

## 为什么需要中文 Skills

- skills.sh 上 99% 是英文 skill，中文开发者的技术栈和部署环境有差异
- 国内项目常用 Vue 3 + Element Plus、微信小程序、阿里云，这些在英文 skill 里覆盖不到
- 中文注释规范、命名习惯、团队协作约定需要专门的指导
- 国内部署（备案、CDN、OSS、RDS）有独特的最佳实践

## 兼容性

这些 skills 遵循 [Agent Skills 开放标准](https://openagentskills.com)，兼容：

- Claude Code / Cursor / Kiro / OpenAI Codex / GitHub Copilot / OpenCode / Aider 等 40+ 工具

## 贡献

欢迎 PR！请参考 [CONTRIBUTING.md](./CONTRIBUTING.md)。

## License

MIT
