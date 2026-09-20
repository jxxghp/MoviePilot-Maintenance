# 运维与 GitHub Actions 部署

## 生产形态

生产只需要 GitHub-hosted runner 和两个目标仓/控制仓的 Actions 配置，不需要本地常驻服务或数据库。每次 workflow 都在临时 runner 中启动一个 Codex CLI 任务；任务结束后 runner 丢弃。

## Workflow 启动前的环境准备

普通 Actions 步骤只做以下准备，不检出目标代码、不编辑目标文件；目标仓库的实际 clone、fetch、checkout 和 GitHub 操作由 Codex CLI 自己完成：

```bash
git config --global user.name "moviepilot-codex[bot]"
git config --global user.email "moviepilot-codex[bot]@users.noreply.github.com"
gh auth setup-git
gh auth status
gh api rate_limit
```

工作流通过 `gh api` 从控制仓 full SHA 读取固定 prompt/schema/config，并把 `moviepilot-telegram` Skill 注入 `.agents/skills`；随后把 `GITHUB_REPOSITORY`、`TARGET_REPOSITORY`、`GITHUB_EVENT_NAME`、`GITHUB_EVENT_PATH`、目标 Issue/PR 编号、目标分支和控制仓 SHA 作为环境变量交给 Codex。提示词要求 Codex 自己执行 `gh repo clone`/`git fetch`/`git checkout`，完成工作后自己 `git commit`、`git push`、`gh pr create` 或 `gh issue comment`。官方 Codex Action 会在非 Git workspace 下以 `--skip-git-repo-check` 启动；Codex 必须立即在 `target/` 中 clone，不能把空 workspace 当作目标代码。Codex 使用完整非交互 permission profile，可进行公网资料检索和 runner 内完整命令执行；`drop-sudo` 只禁止提升为 root，提示词仍禁止执行不可信脚本和上传 Secrets。

## Secrets/Variables

完整的 GitHub 页面配置、文件复制和首次验证步骤见 [`github-configuration.md`](github-configuration.md)。这里保留运行时约定；部署时请按该清单逐项配置。

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `OPENAI_API_KEY` | Repository/organization Secret | 只传给 `openai/codex-action` 的输入 |
| `MAINTENANCE_CONTROL_READ_TOKEN` | 可选 Repository/organization Secret | 仅控制仓改为私有镜像时读取 prompt/schema；不传给 Codex |
| `MAINTENANCE_TARGET_GH_TOKEN` | Repository/organization Secret | 修复/控制仓补漏可选的最小权限 GitHub App/PAT；分析任务不注入 |
| `CODEX_ENABLED` | Repository/organization Variable | 默认 `false`；Secrets/Variables 完整配置并准备启用后才改为 `true` |
| `CODEX_MODEL` | Repository/organization Variable | Codex GPT 模型名 |
| `CODEX_EFFORT` | Repository/organization Variable | 推理强度 |
| `OPENAI_BASE_URL` | Repository/organization Variable | 可选完整 Responses API 地址；例如 `https://mac.jxxghp.cn:8443/v1/responses` |
| `TELEGRAM_BOT_TOKEN` | Repository/organization Secret | Codex 按提示词通知固定 chat |
| `TELEGRAM_CHAT_ID` | Repository/organization Variable/Secret | 固定收件人 |
| `TELEGRAM_ENABLED` | Repository/organization Variable | 默认 `false`；只控制是否向 Codex 注入通知凭据 |

不要设置 `OPENAI_API_KEY`/`CODEX_API_KEY` 为整个 job 环境变量；使用 Codex Action 的专用输入。公开仓库不使用 Codex 登录态文件。

## 权限环境

- `shadow`：`contents: read`、`issues: read`、`pull-requests: read`，不允许 Codex 写入。
- `propose`：只有 `CODEX_PUBLISH_COMMENTS=true` 时才调用 `codex-propose.yml`；给 Codex `issues: write`，允许自己回复分析；不允许 `contents: write`。
- `autofix`：启用 fix dispatch 和 `CODEX_AUTOFIX_ENABLED=true` 后给 Codex `contents: write`、`pull-requests: write`、`issues: write`，只允许主题分支/PR，不允许直接推 `v3` 或 merge。

通过 `concurrency` 按仓库/Issue/PR 串行化重复触发。它不是业务判断器；最新 head/base/Issue 内容由 Codex 每次重新读取。`codex-event.yml`、`codex-propose.yml` 和 `codex-fix.yml` 都不包含 Issue/PR 业务逻辑，区别只在平台权限、permission profile 和提示词入口。

## 事件与安全

目标仓安装 `integrations/MoviePilot-event-bridge.yml`、对应的 `*-codex-run.yml`，以及按需安装 CI/fix 模板：Issue/PR 事件和 CI 完成的第一段 workflow 不接触 OpenAI/Telegram Secret，只用 `GITHUB_TOKEN` 将编号、ref、SHA 转成同仓 `workflow_dispatch`；第二段才调用控制仓 reusable workflow。这样官方 Codex Action 由 `github-actions[bot]` 触发并通过 `allow-bots` 门禁，普通 Issue/PR 作者不会直接消耗 API key。事件桥只使用默认分支 workflow，不 checkout PR 内容。Codex CLI 再自行 clone 目标仓。

`pull_request_target` 只能运行受保护默认分支中的事件桥。分析 prompt 禁止执行 PR 自带 workflow、hooks、postinstall、测试配置和下载脚本；修复 workflow 直接使用已配置的写权限，不等待运行中的人工批准。若平台不允许 `pull_request_target`，使用无 Secrets 的 `pull_request` 或手工 dispatch。

## 停机与恢复

将 Variables 的 `CODEX_ENABLED`、`CODEX_RECONCILE_ENABLED`、`CODEX_AUTOFIX_ENABLED` 和 `CODEX_PUBLISH_COMMENTS` 改为 `false`，撤销不必要的 GitHub token 写权限并取消运行中的 job。失败时保留 Codex 最终输出和 Actions 日志，再从 GitHub 重新触发；不由另一个脚本猜测 Codex 是否已提交，Codex 提示词要求先用 `git status`/`gh pr list` 读回再重试。

## 安装顺序

1. 使用公开的 `jxxghp/MoviePilot-Maintenance` 控制仓，记录提交 full SHA；不要用 floating branch/tag 作为 workflow 或 prompt 来源。
2. 在两个目标仓复制对应的无密钥事件 intake 和 `*-codex-run.yml` trusted run 模板，替换其中的 `CONTROL_REF`；按实际 CI 工作流复制 CI bridge，按需复制 fix dispatch。
3. 在两个目标仓配置 repository/organization Actions Secrets 和 Variables；不需要创建 required-reviewer Environment。完整清单见 `docs/github-configuration.md`。
4. 默认保持 `CODEX_ENABLED=false`、`CODEX_PUBLISH_COMMENTS=false`、`CODEX_AUTOFIX_ENABLED=false`；确认 Secrets/Variables 后将 `CODEX_ENABLED` 改为 `true`，再按需开启自动回复或自动修复，Codex 运行中不会等待人工批准。
5. 先手工触发 Issue/PR 事件验证 artifact，再逐步打开评论和修复入口。整个过程不需要本地服务或本地数据库。
