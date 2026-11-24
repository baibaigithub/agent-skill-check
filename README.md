# Agent Skill Check

简体中文 · [English](README.en.md)

三套面向 AI Agent 的只读审查 Skill。分别用于检查 Skill 的逻辑与结构、审查 Skill 和 Agent 配置的安全性，以及筛查 GitHub 代码仓库的应用与供应链风险。

## 选哪一套

| Skill | 适用场景 | 审查重点 |
| --- | --- | --- |
| [`person-skill-review`](person-skill-review/SKILL.md) | 编写自己的 Skill，或准备改用外部 Skill | 目标与触发条件、输入、判断逻辑、依赖、权限边界、异常处理和验收标准。 |
| [`skill-security-check`](skill-security-check/SKILL.md) | 启用第三方 Skill 前，或对自研 Skill、Plugin、MCP 配置、代理提示词做专项安审 | 提示注入、凭据读取、数据外泄、自动执行、恶意脚本、隐蔽外联及未经验证的依赖。 |
| [`github-repo-security-check`](github-repo-security-check/SKILL.md) | 评估从 GitHub 获取的应用、工具或代码库 | 注入、数据泄露、认证授权、依赖投毒、安装和构建环节、CI 及部署配置。 |

如果代码仓库同时包含会随项目加载、安装或发布的 Agent 资产，对同一提交或本地快照分别做 Skill 安审和代码仓库审查，再合并结论。

## 怎么用

1. 选择需要的 Skill，保留整个目录，包括 `SKILL.md` 和 `references/`。使用支持 `SKILL.md` 的 Agent 时，按该工具的方式添加并调用 Skill；也可以把目录作为审查规则提供给 Agent。
2. 提供待审对象。完整性审查可以从文本或目录开始。安全审查优先提供完整目录，或者提供 GitHub 链接并注明提交、分支和路径范围。
3. 阅读报告中的证据、版本、覆盖范围和盲区。修订后按同一范围复审，发布或安装前核对实际版本。

### 示例请求

- 「用 `person-skill-review` 检查这个 Skill 的触发条件、错误处理和验收标准，给出最小修改建议。」
- 「用 `skill-security-check` 审查这个第三方 Skill 目录，列出数据外发、工具权限和安装前仍需确认的事项。」
- 「用 `github-repo-security-check` 审查这个 GitHub 仓库的指定提交，重点看凭据、越权访问和构建依赖。」

## 审查边界

这些 Skill 用于静态审查，目标代码、安装命令和脚本均按待审材料处理。结论适用于报告中注明的提交或快照与可见范围。只有单个 `SKILL.md`、依赖或二进制无法核验时，报告应明确保留盲区。生产配置、实际网络流量和平台权限需要独立验证。
