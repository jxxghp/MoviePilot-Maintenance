# GitHub 配置清单

这份文档是部署时的逐项清单。项目不需要本地常驻服务、数据库或本地 Agent；需要一个控制仓，以及在 `MoviePilot`、`MoviePilot-Frontend` 两个目标仓安装工作流模板。

## 1. 创建控制仓并取得 full SHA

1. 创建私有仓 `jxxghp/MoviePilot-Maintenance`，把本项目内容推到默认分支。
2. 在控制仓 `Settings → Actions → General` 中允许目标仓使用本仓的 reusable workflows。私有控制仓必须把两个目标仓加入可访问仓库列表。
3. 控制仓提交后取得完整提交号：

   ```bash
   git rev-parse HEAD
   ```

   记下返回的 40 位 `CONTROL_REF`。不要在目标仓模板中使用 `main`、`v3` 或短 SHA。

控制仓中的 `.github/workflows/codex-*.yml` 是唯一执行面；`prompts/`、`schemas/`、`config/codex.toml` 和 `skills/` 都按这个 full SHA 读取。更新控制仓后，重新取得 full SHA，并同步更新目标仓模板中的 `CONTROL_REF`。

## 2. 两个目标仓要复制的文件

目标仓的默认分支是 `v3`。复制后将文件名改成下面的目标路径，并提交到默认分支：

| 源模板 | `jxxghp/MoviePilot` | `jxxghp/MoviePilot-Frontend` |
| --- | --- | --- |
| `integrations/MoviePilot-event-bridge.yml` | `.github/workflows/moviepilot-codex-events.yml` | 使用对应的 Frontend event bridge |
| `integrations/MoviePilot-codex-run.yml` | `.github/workflows/moviepilot-codex-run.yml` | 使用对应的 Frontend codex run |
| `integrations/MoviePilot-ci-bridge.yml` | `.github/workflows/moviepilot-codex-ci.yml` | 同名目标路径 |
| `integrations/MoviePilot-fix-dispatch.yml` | `.github/workflows/moviepilot-codex-fix.yml` | 同名目标路径 |

其中 CI bridge 和 fix dispatch 是可选的；Issue/PR 自动分析至少需要前两份。所有模板中的：

- `jxxghp/MoviePilot-Maintenance`：如果控制仓名称不同，替换成实际 `owner/repository`；
- `CONTROL_REF`：替换成控制仓 40 位 full SHA；
- `CI_WORKFLOW_NAME`：在 CI bridge 中替换成目标仓真实的 CI workflow 名称。

不要把控制仓的 prompt 或 schema 复制到目标仓再自行修改；目标仓只保留触发模板，实际规则统一从控制仓 full SHA 读取。

## 3. 目标仓 Actions 基础设置

在两个目标仓分别打开 `Settings → Actions → General`：

1. 允许使用 Actions 和 reusable workflows；私有控制仓还必须允许它被这两个目标仓访问。
2. `Workflow permissions` 选择 `Read and write permissions`。事件 intake 需要 `actions: write` 调用同仓 `workflow_dispatch`，Codex 回复或修复需要 Issue/PR/Contents 写权限；工作流仍会在 job 级别声明实际使用的权限。
3. 如果希望 Codex 创建 PR，打开允许 GitHub Actions 创建和批准 Pull Request 的仓库选项；不要把 `GITHUB_TOKEN` 手工写入 Secrets。
4. 确认默认分支为 `v3`，并把上述工作流提交到该分支。`pull_request_target` 只会使用默认分支中的 intake 文件。
5. 给 `v3` 配置分支保护：至少禁止直接强推、保留现有 CI 检查。Codex 创建主题分支和 PR，不直接推送 `v3`；是否保留人工 PR review 由维护者自行决定。

当前设计不依赖 GitHub Environment 的 required reviewer。Codex CLI 以非交互方式运行，不会等待人工批准；`shadow`、`propose`、`autofix` 只是工作流变量选择的运行模式，不是需要在运行中点击批准的 Environment。

## 4. 目标仓 Actions Secrets

在 `Settings → Secrets and variables → Actions → Secrets` 中配置。变量放在目标仓或组织级别，不能写进 workflow、prompt、Issue、PR 或 artifact。

| 名称 | 必需 | 配置位置 | 用途 |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` | 是 | 两个目标仓 | 只作为官方 `openai/codex-action` 的 `openai-api-key` 输入 |
| `MAINTENANCE_CONTROL_READ_TOKEN` | 控制仓私有时需要 | 两个目标仓 | 只读控制仓 prompt/schema/config/Skill；不传给 Codex |
| `TELEGRAM_BOT_TOKEN` | 启用 Telegram 时需要 | 两个目标仓 | 注入 Telegram Skill；不要放进 URL 或日志 |
| `MAINTENANCE_TARGET_GH_TOKEN` | 修复/跨仓补漏时可选 | 两个目标仓 | 最小权限 GitHub App 安装 token 或 fine-grained PAT；只传给受保护 fix/reconcile 入口 |

`MAINTENANCE_CONTROL_READ_TOKEN` 只需要 `MoviePilot-Maintenance` 的 Contents read；如果控制仓是公开仓，可以省略。`MAINTENANCE_TARGET_GH_TOKEN` 只授予目标仓所需的 Contents、Issues、Pull requests、Actions 权限，不要授予组织管理权限，也不要把它注入外部 PR 的分析入口。

## 5. 目标仓 Actions Variables

在 `Settings → Secrets and variables → Actions → Variables` 中配置以下名称。没有特殊需求时先使用表中的默认值：

| 名称 | 默认值 | 说明 |
| --- | --- | --- |
| `CODEX_MODEL` | 账号支持的 Codex GPT 模型名 | 传给官方 Codex Action；留空则使用 Action/CLI 默认模型 |
| `CODEX_EFFORT` | `medium` | Codex 推理强度；修复入口模板默认使用 `high` |
| `OPENAI_BASE_URL` | 空 | OpenAI 官方地址留空；兼容代理才配置自定义地址 |
| `TELEGRAM_ENABLED` | `false` | 是否向 Codex 注入 Telegram 通知凭据 |
| `TELEGRAM_CHAT_ID` | 空 | 固定维护者用户或群组 ID，不能由 Issue/PR/模型提供 |
| `CODEX_PUBLISH_COMMENTS` | `false` | `false` 只上传结果 artifact；`true` 才允许 Codex 用 `gh issue comment` 回复 |
| `CODEX_AUTOFIX_ENABLED` | `false` | 只有 fix dispatch 使用；为 `true` 时 Codex 才能在有写权限的入口创建分支、提交和 PR |

推荐的初始配置是：`CODEX_PUBLISH_COMMENTS=false`、`CODEX_AUTOFIX_ENABLED=false`、`TELEGRAM_ENABLED=false`。这不会限制 Codex CLI 的本地执行权限，只限制它是否拥有对应的 GitHub 写入能力和是否主动发送通知。

## 6. 控制仓补漏 workflow 的配置

如果启用 `.github/workflows/codex-reconcile.yml`，在控制仓配置：

| 名称 | 类型 | 用途 |
| --- | --- | --- |
| `OPENAI_API_KEY` | Secret | 调用 Codex Action |
| `MAINTENANCE_TARGET_GH_TOKEN` | Secret | 读取/操作目标仓，按需授予最小权限 |
| `TELEGRAM_BOT_TOKEN` | Secret | 可选通知 |
| `CODEX_MODEL` | Variable | Codex GPT 模型名 |
| `CODEX_EFFORT` | Variable | 推理强度 |
| `OPENAI_BASE_URL` | Variable | 可选模型代理地址 |
| `TELEGRAM_ENABLED` | Variable | `true`/`false` |
| `TELEGRAM_CHAT_ID` | Variable 或 Secret | 固定收件人 |
| `DEFAULT_TARGET_REPOSITORY` | Variable | 默认 `jxxghp/MoviePilot` |

补漏 workflow 不依赖目标仓的 `GITHUB_TOKEN`，因为它在控制仓运行；必须配置 `MAINTENANCE_TARGET_GH_TOKEN` 才能跨仓读取目标项目。

## 7. Telegram 配置

1. 在 Telegram 创建 Bot，取得 Bot token，写入 `TELEGRAM_BOT_TOKEN`。
2. 将维护者用户或群组的固定 chat ID 写入 `TELEGRAM_CHAT_ID`。
3. 将 `TELEGRAM_ENABLED` 改为 `true`。

消息格式、长度、脱敏、失败语义和“只向固定收件人发送”由注入的 `skills/moviepilot-telegram/SKILL.md` 规定。Telegram Skill 不保存 token/chat ID，也不接收 Issue/PR 中提供的收件人。

## 8. Codex CLI 权限和 GitHub 权限的区别

Codex Action 使用控制仓 `config/codex.toml` 中的 `moviepilot-autonomous-net` profile：它基于官方 `:danger-full-access`，允许 Codex CLI 在 runner 上完整执行 clone、fetch、checkout、编辑、测试、commit、push、PR/comment 和公开资料检索，不产生交互式授权请求；`drop-sudo` 只禁止提升为 root。GitHub API 能做什么仍由当前 workflow 的 `permissions` 和 `GH_TOKEN` 决定，二者不是同一层权限。

因此：

- `gh` 不需要额外 Skill；Actions 自动提供的 `GITHUB_TOKEN` 经 `gh auth setup-git` 使用；
- Issue/PR/CI 的事实判断、是否改代码、是否符合方向、是否回复、如何修复以及所有 Git 操作都交给 Codex CLI；
- Actions 只转发事件、准备凭据、注入 prompt/Skill/config、启动 Codex、保存最终 JSON 和根据退出码结束；
- 不要在 runner 上安装第二套 Agent、GitHub API 客户端、补丁应用器或数据库。

## 9. 首次验证顺序

1. 两个目标仓先保持 `CODEX_PUBLISH_COMMENTS=false`、`CODEX_AUTOFIX_ENABLED=false`、`TELEGRAM_ENABLED=false`。
2. 提交模板后打开一个测试 Issue，确认 `moviepilot-codex-events.yml` 只做一次同仓 `workflow_dispatch`，然后 `moviepilot-codex-run.yml` 启动 Codex。
3. 检查 Actions 日志中的 `gh auth status`、控制仓 full SHA、Codex 最终 JSON artifact 和目标仓/目标编号；确认没有 `actions/checkout`，目标代码由 Codex 自己 clone。
4. 确认稳定后再把 `CODEX_PUBLISH_COMMENTS` 改为 `true`；需要自动修复时再启用 fix dispatch 和 `CODEX_AUTOFIX_ENABLED=true`。
5. 修改控制仓 prompt、Skill、Schema 或权限后，重新提交控制仓、更新所有目标仓模板的 `CONTROL_REF`，再重复验证。

官方参考：

- [Reusable workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
- [Environment secrets 与保护规则](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [Codex GitHub Action](https://learn.chatgpt.com/docs/github-action)
- [Codex permissions](https://learn.chatgpt.com/docs/permissions)
