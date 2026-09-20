# MoviePilot-Maintenance

用 GitHub Actions 模拟维护者使用 Codex CLI 维护 MoviePilot 后端和前端。

本仓库不实现 Agent、Issue/PR 分析器或修复器。它只提供：

- 目标仓事件桥和可复用 Actions workflow；
- Codex CLI 的固定提示词、输出 Schema 和任务上下文约定；
- 可注入的 `moviepilot-telegram` Skill，用于固定格式、固定收件人和脱敏通知；
- `gh`/`git` 环境准备、权限边界、Actions Secrets/Variables 和 Telegram 接线；
- shadow、分析、Codex 自主修复/提交/建 PR、交付跟踪的 workflow 模板。

唯一目标是把 GitHub Actions 配成维护者的 Codex CLI 工作台：平台只负责事件转发、权限、环境、Skill 和最终输出，所有 Issue/PR 判断、资料检索、代码库操作、答复与修复都交给 Codex CLI。

每次任务都由官方 `openai/codex-action@v1` 调用 `codex exec`。目标代码的 clone、fetch、checkout、分支、编辑、测试、commit、push、PR、Issue 回复和 Telegram 通知（如启用）均由 Codex CLI 根据提示词完成；Actions 只负责启动、提供环境和根据 Codex 退出码收尾。

`gh` 授权使用 Actions 的 `GITHUB_TOKEN` 和 `gh auth setup-git`，不需要另做 GitHub Skill；只有控制仓定时跨仓补漏才使用单独的最小权限 GitHub App/PAT Secret。Telegram Skill 只约束消息，不保存任何凭据。

部署与平台配置见 [`docs/operations.md`](docs/operations.md)，workflow 角色和自动触发链见 [`docs/workflows.md`](docs/workflows.md)，逐项配置清单见 [`docs/github-configuration.md`](docs/github-configuration.md)，凭据见 [`docs/credentials.md`](docs/credentials.md)，分阶段目标见 [`docs/roadmap.md`](docs/roadmap.md)。
