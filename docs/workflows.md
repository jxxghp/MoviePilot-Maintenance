# Workflow 运行链路

本项目有两类 workflow：目标仓中的“入口/转发” workflow，以及控制仓中的“Codex 执行” reusable workflow。目标仓入口只负责监听 GitHub 事件、转发可信的编号/ref/SHA；真正的 Issue/PR 判断、资料检索、代码检出、修改、测试、提交、建 PR 和回复都由控制仓 full SHA 中的 Codex CLI 完成。

## 先看结论

Issue 或 PR 的自动入口是目标仓的 `.github/workflows/moviepilot-codex-events.yml`：

```text
Issue opened/reopened/edited
PR opened/reopened/synchronize/ready_for_review/edited
        │
        ▼
moviepilot-codex-events.yml       自动监听；无 OpenAI/Telegram Secret
        │ workflow_dispatch
        ▼
moviepilot-codex-run.yml          同仓分发；按变量选择 shadow 或 propose
        │ reusable workflow_call
        ├── control/codex-event.yml    只读分析，保存 artifact
        └── control/codex-propose.yml  可回复 Issue/PR，不改代码
```

CI 失败的自动入口是目标仓的 `.github/workflows/moviepilot-codex-ci.yml`，它只在配置的 CI workflow 以 `failure` 结束时进入上面同一条 `moviepilot-codex-run.yml` 链路。

自动修复不是 Issue/PR 事件的默认后续步骤。需要维护者手工 dispatch 目标仓的 `.github/workflows/moviepilot-codex-fix.yml`，并且 `CODEX_ENABLED=true` 与 `CODEX_AUTOFIX_ENABLED=true` 同时满足，才会调用有写权限的 `control/codex-fix.yml`。

## 目标仓中的四个文件

以下文件需要分别复制到 `MoviePilot` 和 `MoviePilot-Frontend` 的 `.github/workflows/`。Frontend 使用带 `Frontend` 前缀的对应模板。

| 目标仓文件 | 触发方式 | 作用 | 是否直接调用 Codex | 写权限 |
| --- | --- | --- | --- | --- |
| `moviepilot-codex-events.yml` | 自动：Issue `opened/reopened/edited`；`pull_request_target` 的 `opened/reopened/synchronize/ready_for_review/edited` | 把事件转换为同仓 `workflow_dispatch`，只转发事件类型、编号、ref、SHA | 否 | 无代码写权限 |
| `moviepilot-codex-run.yml` | 由上一个文件或 CI bridge 自动 dispatch | 按 `CODEX_ENABLED`、`CODEX_PUBLISH_COMMENTS` 选择控制仓的 shadow/propose reusable workflow | 间接 | shadow 只读；propose 只能评论 |
| `moviepilot-codex-ci.yml` | 自动：配置的 CI workflow `completed` 且结论为 `failure` | 把失败 run ID、head SHA 和分支转发给 `moviepilot-codex-run.yml` | 否 | 无代码写权限 |
| `moviepilot-codex-fix.yml` | 仅 `workflow_dispatch` 手工触发 | 经过两个变量门禁后调用有写权限的修复 reusable workflow | 间接 | 允许主题分支、commit、push、PR 和 Issue 回复；不直接推 `v3`，不自动 merge |

事件 bridge 和 CI bridge 不读取 `OPENAI_API_KEY`、`TELEGRAM_BOT_TOKEN`，也不 checkout 目标代码。它们只使用目标仓自动提供的 `GITHUB_TOKEN` 调用同仓 `gh workflow run`。

## 控制仓中的四个 reusable workflow

这些文件不能作为目标仓的 GitHub 事件入口；它们由目标仓 workflow 通过固定的 40 位 `CONTROL_REF` 调用。

| 控制仓文件 | 被谁调用 | 权限边界 | 主要职责 |
| --- | --- | --- | --- |
| `.github/workflows/codex-event.yml` | `moviepilot-codex-run.yml` 的 `shadow` job | `contents/issues/pull-requests: read` | 读取上下文，调用官方 Codex CLI，保存结构化分析结果 artifact |
| `.github/workflows/codex-propose.yml` | `moviepilot-codex-run.yml` 的 `propose` job | `contents: read`、`issues: write`、`pull-requests: read` | 让 Codex 自己判断并通过 `gh` 回复 Issue/PR；不改代码 |
| `.github/workflows/codex-fix.yml` | 目标仓 `moviepilot-codex-fix.yml` | `contents/issues/pull-requests: write` | 让 Codex 自己检出、修改、测试、commit、push、建 PR 和回写结果 |
| `.github/workflows/codex-reconcile.yml` | 控制仓定时任务或手工 dispatch；默认关闭 | 由控制仓配置的跨仓 token 决定 | 可选的定时交付/补漏任务，不属于 Issue/PR 自动入口 |

控制仓的 `.github/workflows/contract-validation.yml` 只校验 prompt、Schema、权限和 workflow 引用；它不是 MoviePilot Issue/PR 维护入口。

## 变量决定自动链路的行为

事件 bridge 即使被触发，也不会绕过这些门禁：

| 条件 | 实际行为 |
| --- | --- |
| `CODEX_ENABLED=false` | `moviepilot-codex-run.yml` 的 Codex job 跳过；不会调用模型 |
| `CODEX_ENABLED=true` 且 `CODEX_PUBLISH_COMMENTS=false` | 进入 `codex-event.yml`，只读分析并保存 artifact |
| `CODEX_ENABLED=true` 且 `CODEX_PUBLISH_COMMENTS=true` | 进入 `codex-propose.yml`，Codex 可自行决定是否回复 Issue/PR，但仍不能改代码 |
| `CODEX_ENABLED=true` 且 `CODEX_AUTOFIX_ENABLED=true` | 手工 `moviepilot-codex-fix.yml` 才能进入 `codex-fix.yml`；自动 Issue/PR 事件不会隐式升级为写权限 |
| `TELEGRAM_ENABLED=true` 且 `TELEGRAM_CHAT_ID` 非空 | propose/fix 才会向 Codex 注入固定 Telegram 收件人和 Skill；缺少 chat ID 自动关闭通知 |

因此推荐的启用顺序是：先只读 shadow，再打开评论，最后在确认 API、权限和分支保护后手工启用 fix。`CODEX_AUTOFIX_ENABLED` 不会让 Issue/PR 自动事件获得写权限。

## 手工修复示例

确认变量已配置后，维护者可以手工启动某个 Issue 的修复入口：

```bash
gh workflow run moviepilot-codex-fix.yml \
  --repo jxxghp/MoviePilot \
  --ref v3 \
  -f issue_number=6738 \
  -f prompt_name=issue-fix
```

这条命令只启动 workflow；后续代码检出、是否修复、测试、提交、PR 和 Issue 回复均由 Codex CLI 按控制仓 prompt 完成。运行结束后应查看 Actions run、Codex result artifact、目标分支/PR 和 Issue 回读状态。
