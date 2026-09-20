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

工作流通过 `gh api` 从控制仓 full SHA 读取固定 prompt/schema/config，并把 `moviepilot-telegram` Skill 注入 `.agents/skills`；随后把 `GITHUB_REPOSITORY`、`TARGET_REPOSITORY`、`GITHUB_EVENT_NAME`、`GITHUB_EVENT_PATH`、目标 Issue/PR 编号、目标分支和控制仓 SHA 作为环境变量交给 Codex。提示词要求 Codex 自己执行 `gh repo clone`/`git fetch`/`git checkout`，完成工作后自己 `git commit`、`git push`、`gh pr create` 或 `gh issue comment`。官方 Codex Action 会在非 Git workspace 下以 `--skip-git-repo-check` 启动；Codex 必须立即在 `target/` 中 clone，不能把空 workspace 当作目标代码。permission profile 允许公网资料检索和公共依赖查询，但仍使用 `drop-sudo`、工作区边界和平台的本地/私网保护；提示词禁止执行不可信脚本和上传 Secrets。

## Secrets/Variables

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `OPENAI_API_KEY` | Environment Secret | 只传给 `openai/codex-action` 的输入 |
| `MAINTENANCE_CONTROL_READ_TOKEN` | Environment Secret | 读取私有控制仓 prompt/schema；不传给 Codex |
| `MAINTENANCE_TARGET_GH_TOKEN` | Environment Secret | 修复/控制仓补漏可选的最小权限 GitHub App/PAT；分析任务不注入 |
| `CODEX_MODEL` | Environment Variable | Codex GPT 模型名 |
| `CODEX_EFFORT` | Environment Variable | 推理强度 |
| `OPENAI_BASE_URL` | Environment Variable | 可选 API 代理地址 |
| `TELEGRAM_BOT_TOKEN` | Environment Secret | Codex 按提示词通知固定 chat |
| `TELEGRAM_CHAT_ID` | Environment Variable/Secret | 固定收件人 |
| `TELEGRAM_ENABLED` | Environment Variable | 默认 `false`；只控制是否向 Codex 注入通知凭据 |

不要设置 `OPENAI_API_KEY`/`CODEX_API_KEY` 为整个 job 环境变量；使用 Codex Action 的专用输入。公开仓库不使用 Codex 登录态文件。

## 权限环境

- `shadow`：`contents: read`、`issues: read`、`pull-requests: read`，不允许 Codex 写入。
- `propose`：只有 `CODEX_PUBLISH_COMMENTS=true` 时才调用 `codex-propose.yml`；在 required reviewer 批准后给 Codex `issues: write`，允许自己回复分析；不允许 `contents: write`。
- `autofix`：required reviewer 批准后给 Codex `contents: write`、`pull-requests: write`、`issues: write`，只允许主题分支/PR，不允许直接推 `v3` 或 merge。

通过 `concurrency` 按仓库/Issue/PR 串行化重复触发。它不是业务判断器；最新 head/base/Issue 内容由 Codex 每次重新读取。`codex-event.yml`、`codex-propose.yml` 和 `codex-fix.yml` 都不包含 Issue/PR 业务逻辑，区别只在平台权限、sandbox 和提示词入口。

## 事件与安全

目标仓安装 `integrations/MoviePilot-event-bridge.yml`、对应的 `*-codex-run.yml`，以及按需安装 CI/fix 模板：Issue/PR 事件和 CI 完成的第一段 workflow 不接触 OpenAI/Telegram Secret，只用 `GITHUB_TOKEN` 将编号、ref、SHA 转成同仓 `workflow_dispatch`；第二段才调用控制仓 reusable workflow。这样官方 Codex Action 由 `github-actions[bot]` 触发并通过 `allow-bots` 门禁，普通 Issue/PR 作者不会直接消耗 API key。事件桥只使用默认分支 workflow，不 checkout PR 内容。Codex CLI 再自行 clone 目标仓。

`pull_request_target` 只能运行受保护默认分支中的事件桥。分析 prompt 禁止执行 PR 自带 workflow、hooks、postinstall、测试配置和下载脚本；修复 workflow 必须人工批准并使用独立 Environment。若平台不允许 `pull_request_target`，使用无 Secrets 的 `pull_request` 或手工 dispatch。

## 停机与恢复

将 Environment 的 `CODEX_AUTOFIX_ENABLED`/`CODEX_PUBLISH_COMMENTS` 改为 `false`，撤销 required reviewer 以外的写权限并取消运行中的 job。失败时保留 Codex 最终输出和 Actions 日志，人工从 GitHub 重新触发；不由另一个脚本猜测 Codex 是否已提交，Codex 提示词要求先用 `git status`/`gh pr list` 读回再重试。

## 安装顺序

1. 创建私有控制仓，提交本项目并记录提交 full SHA；不要用 floating branch/tag 作为 workflow 或 prompt 来源。
2. 在两个目标仓复制对应的事件桥模板，替换 `CONTROL_REF`；按实际 CI 工作流复制 CI bridge。
3. 为目标仓创建 `shadow`、`propose`、`autofix` Environment；在 `propose`/`autofix` 配置 required reviewers 和最小权限。
4. 在 Environment 中配置 `OPENAI_API_KEY`、模型变量、可选代理地址、控制仓读取 token 和固定 Telegram 收件人；默认保持 `CODEX_PUBLISH_COMMENTS=false`、`CODEX_AUTOFIX_ENABLED=false`。
5. 先手工触发 Issue/PR 事件验证 artifact，再逐步打开评论和修复入口。整个过程不需要本地服务或本地数据库。
