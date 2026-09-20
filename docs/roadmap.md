# MoviePilot-Maintenance 路线图

本文件保存长期目标的执行边界。唯一的执行器是 GitHub Actions 中的官方 Codex CLI；本仓库不发展为 Agent 平台。

## 当前唯一目标

创建一个只包含 GitHub Actions、Codex CLI 配置、可信提示词、可注入 Skill 和运行权限的控制仓。收到 Issue、PR 或 CI 事件后，Actions 只传递上下文并启动 Codex；问题是否成立、是否需要改代码、是否符合 MoviePilot 方向、PR 是否适合合并、检出/修改/测试/提交/回复/通知全部由 Codex CLI 完成。

## P0 — Actions 与 Codex CLI 环境（active）

- 创建控制仓 workflow、事件桥模板、提示词和 Codex 输出 Schema。
- 通过 `gh auth setup-git`、`git config`、`GITHUB_EVENT_PATH`、`GITHUB_REPOSITORY` 等准备 Codex 的模拟维护者环境。
- Codex 自己 clone/fetch/checkout 目标仓，不使用普通 Actions checkout。
- Codex 自己判断 Issue 是否成立、是否需要修改、是否符合项目方向、PR 是否适合合并以及是否需要回复；控制仓不实现这些业务逻辑。
- 默认 shadow，只输出结果，不评论、不提交、不建 PR。
- Issue/PR/CI intake 先用无 OpenAI Secret 的同仓 `workflow_dispatch` 转发，再由 `github-actions[bot]` 启动官方 Codex Action，避免普通外部参与者直接消耗 API key。

## P1 — Issue/PR/CI 分析（planned）

Codex 读取 Issue/PR/diff/CI，自己使用 `gh` 回复中文分析；Actions 仅检查 Codex 退出码和保存最终输出。外部 fork、Secrets 和 `pull_request_target` 依赖 GitHub 平台安全策略。

## P2 — Codex 自主修复和提交（planned）

维护者批准后，Codex 自己创建主题分支、编辑代码、运行测试、commit、push 和创建 PR；工作流不应用补丁、不创建第二个代码代理。默认不允许合并主分支。

## P3 — 交付与发布跟踪（planned）

Codex 自己读取最终 CI、前端 Release、后端发布和 Issue 状态，按提示词回复/关闭或保持开放；Actions 只提供权限和收尾。

## P4 — 运营与扩展（planned）

补漏、预算、Telegram 通知、凭据轮换、kill switch、历史样本回放和维护者手册。任何新能力仍通过提示词和 workflow 权限增加，不引入自研 Agent。

## 外部平台安装清单

维护者需要在 GitHub 配置：控制仓权限、目标仓事件桥、Actions Environments、`OPENAI_API_KEY`、控制仓读取 token、可选跨仓 `MAINTENANCE_TARGET_GH_TOKEN`、`CODEX_MODEL`/`CODEX_EFFORT`/`OPENAI_BASE_URL`、`TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID`、required reviewers、分支保护和默认 full-SHA control ref。代码不会伪造这些平台状态。
