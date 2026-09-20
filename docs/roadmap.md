# MoviePilot-Maintenance 路线图

本文件保存长期目标的执行边界。唯一的执行器是 GitHub Actions 中的官方 Codex CLI；本仓库不发展为 Agent 平台。

## 当前唯一目标

创建一个只包含 GitHub Actions、Codex CLI 配置、可信提示词、可注入 Skill 和运行权限的控制仓。收到 Issue、PR 或 CI 事件后，Actions 只传递上下文并启动 Codex；问题是否成立、是否需要改代码、是否符合 MoviePilot 方向、PR 是否适合合并、检出/修改/测试/提交/回复/通知全部由 Codex CLI 完成。

## P0 — Actions 与 Codex CLI 环境（接线完成，endpoint 与模型路由已验证）

- 创建控制仓 workflow、事件桥模板、提示词和 Codex 输出 Schema。
- 通过 `gh auth setup-git`、`git config`、`GITHUB_EVENT_PATH`、`GITHUB_REPOSITORY` 等准备 Codex 的模拟维护者环境；使用完整非交互 permission profile，不等待人工授权。
- Codex 自己 clone/fetch/checkout 目标仓，不使用普通 Actions checkout。
- Codex 自己判断 Issue 是否成立、是否需要修改、是否符合项目方向、PR 是否适合合并以及是否需要回复；控制仓不实现这些业务逻辑。
- 默认 shadow，只输出结果，不评论、不提交、不建 PR。
- Issue/PR/CI intake 先用无 OpenAI Secret 的同仓 `workflow_dispatch` 转发，再由 `github-actions[bot]` 启动官方 Codex Action，避免普通外部参与者直接消耗 API key。

当前证据：控制仓契约校验已成功；两个目标仓的 workflow 已实际进入官方 `codex exec`，endpoint 预检从 GitHub-hosted runner 获得预期 HTTP 响应，配置的模型为 `gpt-5.6-luna`、推理强度为 `max`。真实任务的结构化结果、代码提交和 PR 仍须以每次运行的最终状态为准，不能用 endpoint 连通性代替业务验收。

## P1 — Issue/PR/CI 分析与自动答复（实现完成，验收待历史回放与终态样本）

Codex 读取 Issue/PR/diff/CI，自己判断问题、项目方向、PR 合并适配性和回复内容，并按固定提示词使用 `gh` 回复中文分析；Actions 仅检查 Codex 退出码和保存最终输出。外部 fork、Secrets 和 `pull_request_target` 仍受 GitHub 平台安全策略约束，但 Codex 运行过程不等待人工授权。

API endpoint 和模型路由已恢复；历史样本 30 例回放、评论内容复核、提示词注入样例以及至少一个终态真实分析任务尚未完成，不能把分析接线的存在视为分析质量验收。

## P2 — Codex 自主修复和提交（实现完成，验收待 API 额度与低风险样本）

当预先配置的 `CODEX_AUTOFIX_ENABLED=true` 且入口具备写权限时，Codex 自己判断是否修复，创建主题分支、编辑代码、运行测试、commit、push 和创建 PR；工作流不应用补丁、不创建第二个代码代理。运行中不等待人工批准，直接推送到 `v3` 和合并仍不在当前提示词的默认动作内。当前已增加目标仓和 reusable workflow 两级写权限门禁。

10 个低风险 Bug 的实际修复、测试、候选 PR 和回读验收尚未完成。

## P3 — 交付与发布跟踪（接线完成，运行验收待补）

`delivery` prompt 与 scheduled reconcile workflow 让 Codex 自己读取最终 CI、前端 Release、后端发布和 Issue 状态，决定回复、关闭或保持开放；Actions 只提供权限和收尾，不实现业务判断。

## P4 — 运营与扩展（文档与安全约束已覆盖，持续运营项）

补漏、预算、Telegram 通知、凭据轮换、kill switch、历史样本回放和维护者手册。任何新能力仍通过提示词和 workflow 权限增加，不引入自研 Agent。

## 外部平台安装清单

维护者需要在 GitHub 配置：控制仓权限、受保护的 `main` 控制分支、目标仓事件桥、Actions Secrets/Variables、`OPENAI_API_KEY`、控制仓读取 token、可选跨仓 `MAINTENANCE_TARGET_GH_TOKEN`、`CODEX_MODEL`/`CODEX_EFFORT`/完整 Responses API endpoint、`TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` 和分支保护。代码不会伪造这些平台状态。
