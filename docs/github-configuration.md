# GitHub 配置清单

这份文档是部署时的逐项清单。项目不需要本地常驻服务、数据库或本地 Agent；需要一个控制仓，以及在 `MoviePilot`、`MoviePilot-Frontend` 两个目标仓安装工作流模板。

四个模板不是同一类入口：Issue/PR 的自动入口是 `moviepilot-codex-events.yml`，CI 失败的自动入口是 `moviepilot-codex-ci.yml`，`moviepilot-codex-run.yml` 负责二段式分发，`moviepilot-codex-fix.yml` 只在维护者手工 dispatch 且两个开关都打开时获得写权限。完整链路和权限表见 [`workflows.md`](workflows.md)。

## 1. 创建控制仓并跟随最新 `main`

1. 使用公开仓 `jxxghp/MoviePilot-Maintenance`，把本项目内容推到默认分支。
2. 在控制仓 `Settings → Actions → General` 中确认允许 Actions 和 reusable workflows。公开控制仓不需要额外配置目标仓访问列表。
3. 目标仓模板默认使用控制仓 `main`：`uses: ...@main` 且 `control_ref: main`。这样控制仓提交后，下一次目标仓运行自动读取最新的 prompt/schema/config/Skill，不需要再改目标仓 workflow。
4. 为了保留可审计性，保护控制仓 `main`，至少要求 PR review、契约校验通过，并限制能够修改 `.github/workflows/`、`prompts/`、`schemas/` 和 `skills/` 的人员。若某次高风险任务需要不可变证据，仍可把两个目标仓临时改为 full SHA。

## 2. 两个目标仓要复制的文件

目标仓的默认分支是 `v3`。复制后将文件名改成下面的目标路径，并提交到默认分支：

| 源模板 | `jxxghp/MoviePilot` | `jxxghp/MoviePilot-Frontend` |
| --- | --- | --- |
| `integrations/MoviePilot-event-bridge.yml` | `.github/workflows/moviepilot-codex-events.yml` | 使用对应的 Frontend event bridge |
| `integrations/MoviePilot-codex-run.yml` | `.github/workflows/moviepilot-codex-run.yml` | 使用对应的 Frontend codex run |
| `integrations/MoviePilot-ci-bridge.yml` | `.github/workflows/moviepilot-codex-ci.yml` | 后端 CI 失败入口 |
| `integrations/MoviePilot-Frontend-ci-bridge.yml` | `.github/workflows/moviepilot-codex-ci.yml` | 前端 CI 失败入口 |
| `integrations/MoviePilot-fix-dispatch.yml` | `.github/workflows/moviepilot-codex-fix.yml` | 同名目标路径 |

其中 CI bridge 和 fix dispatch 是可选的；Issue/PR 自动分析至少需要前两份。所有模板中的：

- `jxxghp/MoviePilot-Maintenance`：如果控制仓名称不同，替换成实际 `owner/repository`；
- `CONTROL_REF`：保持为 `main`；只有需要临时冻结版本时才替换成 40 位 full SHA；
- `CI_WORKFLOW_NAME`：在 CI bridge 中替换成目标仓真实的 CI workflow 名称。

不要把控制仓的 prompt 或 schema 复制到目标仓再自行修改；目标仓只保留触发模板，实际规则统一从受保护控制仓 `main` 读取。

## 3. 目标仓 Actions 基础设置

在两个目标仓分别打开 `Settings → Actions → General`：

1. 允许使用 Actions 和 reusable workflows；公开控制仓不需要额外授予目标仓访问权限。
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
| `MAINTENANCE_CONTROL_READ_TOKEN` | 可选 | 两个目标仓 | 仅控制仓改为私有镜像时读取 prompt/schema/config/Skill；不传给 Codex |
| `TELEGRAM_BOT_TOKEN` | 启用 Telegram 时需要 | 两个目标仓 | 注入 Telegram Skill；不要放进 URL 或日志 |
| `MAINTENANCE_TARGET_GH_TOKEN` | 修复/跨仓补漏时可选 | 两个目标仓 | 最小权限 GitHub App 安装 token 或 fine-grained PAT；只传给受保护 fix/reconcile 入口 |

公开控制仓可以省略 `MAINTENANCE_CONTROL_READ_TOKEN`；如果维护者建立私有控制仓镜像，才为该镜像配置 Contents read token。`MAINTENANCE_TARGET_GH_TOKEN` 只授予目标仓所需的 Contents、Issues、Pull requests、Actions 权限，不要授予组织管理权限，也不要把它注入外部 PR 的分析入口。

## 5. 目标仓 Actions Variables

在 `Settings → Secrets and variables → Actions → Variables` 中配置以下名称。没有特殊需求时先使用表中的默认值：

| 名称 | 默认值 | 说明 |
| --- | --- | --- |
| `CODEX_ENABLED` | `false` | 总开关；确认 API key、模型和权限已配置后才改为 `true` |
| `CODEX_MODEL` | 账号支持的 Codex GPT 模型名（当前推荐 `gpt-5.6-luna`） | 传给官方 Codex Action；必须与账户和中转服务实际支持的模型一致 |
| `CODEX_EFFORT` | `max` | Codex 推理强度；当前维护配置使用 `max`，模板缺省也保持为 `max` |
| `OPENAI_BASE_URL` | 空 | OpenAI 官方地址留空；兼容代理配置完整 Responses API 地址，例如 `https://codex-api.example.com/v1/responses`。推荐使用 Cloudflare Tunnel 或其它公网双栈 HTTPS 中转；不要把只在本机 DNS/IPv6 网络可达的地址直接给 GitHub-hosted runner |
| `TELEGRAM_ENABLED` | `false` | 是否向 Codex 注入 Telegram 通知凭据 |
| `TELEGRAM_CHAT_ID` | 空 | 固定维护者用户或群组 ID，不能由 Issue/PR/模型提供 |
| `CODEX_PUBLISH_COMMENTS` | `false` | `false` 只上传结果 artifact；`true` 才允许 Codex 用 `gh issue comment` 回复 |
| `CODEX_AUTOFIX_ENABLED` | `false` | 只有 fix dispatch 使用；为 `true` 时 Codex 才能在有写权限的入口创建分支、提交和 PR |

推荐的初始配置是：`CODEX_ENABLED=false`、`CODEX_PUBLISH_COMMENTS=false`、`CODEX_AUTOFIX_ENABLED=false`、`TELEGRAM_ENABLED=false`。确认 `OPENAI_API_KEY` 和其它配置就绪后才把 `CODEX_ENABLED` 改为 `true`。这不会限制 Codex CLI 的本地执行权限，只控制是否启动任务以及是否拥有对应的 GitHub 写入能力和是否主动发送通知。

如果配置了 `OPENAI_BASE_URL`，每次 Codex job 会先执行不携带 API key 的 HTTPS 连通性预检；预检失败时不会启动 Codex，也不会消耗模型请求。日志中的 `remote=... status=...` 用于确认 runner 实际访问的地址；`status=000` 表示尚未建立 HTTP 连接，`5xx` 表示中转或源站不可用，工作流会直接停止。若源站只有 IPv6，不能仅凭本机 `curl -6` 成功判断 GitHub-hosted runner 可达；应使用下面的 Cloudflare Tunnel 中转，或改用具备 IPv6 出口的 self-hosted runner。

## 6. 用 Cloudflare Tunnel 中转 IPv6 源站（推荐）

如果 MoviePilot 的 Responses API 只在 Mac 的公网 IPv6 上提供服务，推荐使用 Cloudflare Tunnel。Tunnel 由 Mac 主动向 Cloudflare 建立出站连接，GitHub Actions 只访问 Cloudflare 公网边缘，不需要 GitHub-hosted runner 直接访问你的 IPv6 源站，也不需要在路由器上开放 8443 入站。

### 6.1 在 Cloudflare 创建 Tunnel

1. 把一个域名接入 Cloudflare，在 Zero Trust/Networks/Tunnels 创建一个 `cloudflared` Tunnel。
2. 在 Tunnel 的 Published application 中新增公共主机名，例如 `codex-api.example.com`。
3. 如果 `cloudflared` 与 API 在同一台 Mac，Service URL 填 `https://localhost:8443`。不要把公共主机名再填回 Service URL，否则会形成回环。
4. 在 Additional application settings 中配置：
   - `Origin Server Name`：`mac.jxxghp.cn`，因为现有源站证书签发给这个名称；
   - `Disable TLS verification`：关闭；
   - `HTTP Host Header`：如果源站按 Host 路由，填 `mac.jxxghp.cn`；否则可留空。
5. 在 Mac 安装并运行 Cloudflare 给出的 connector 命令。Homebrew 方式示例：

   ```bash
   brew install cloudflared
   sudo cloudflared service install '<从 Cloudflare 控制台复制的 Tunnel token>'
   ```

   Tunnel token 只保存在 Mac 的服务配置中，不要提交到仓库、Issue、PR 或 GitHub Actions Variables。

Cloudflare 会自动为 `codex-api.example.com` 创建指向 `<TUNNEL_ID>.cfargotunnel.com` 的 DNS 记录。保持代理状态为橙云；不要把这个记录改成 DNS only。Cloudflare 官方文档说明，使用 HTTPS 源站时应通过 `Origin Server Name` 保持证书校验，而不是关闭 TLS 校验。

如果公共入口返回 `526`，说明请求已经到达 Cloudflare，但 Cloudflare 无法通过源站 TLS 证书校验。Tunnel 模式应检查 `Service URL=https://localhost:8443`、`Origin Server Name=mac.jxxghp.cn` 和关闭状态的 `Disable TLS verification`；直接橙云代理模式则需要让源站证书覆盖 `codex-api.example.com`，或改用 Cloudflare Origin CA 证书，并保持 SSL/TLS 为 `Full (strict)`。不要用 `Flexible` 规避证书错误。

### 6.2 验证并切换 Actions

先从一台不在同一局域网的机器验证公共入口：

```bash
curl -i --max-time 15 https://codex-api.example.com/v1/responses
```

不带 API key 时得到 `401` 或接口定义的 `4xx` 是正常的；重点是不能出现 DNS、TCP、TLS 或 `502/504` 错误。然后在 `MoviePilot` 和 `MoviePilot-Frontend` 两个目标仓的 Actions Variables 中设置：

```text
OPENAI_BASE_URL=https://codex-api.example.com/v1/responses
```

不要修改 `OPENAI_API_KEY` 的配置方式。下一次运行日志应出现类似 `Responses endpoint preflight: remote=<Cloudflare address> status=401`，随后才会启动 Codex CLI。

不要给这个主机名套 Cloudflare Access 的网页登录策略；官方 Codex Action 不会进行浏览器登录。API 自身的 `OPENAI_API_KEY`、Cloudflare WAF/rate limit 和源站应用鉴权可以继续使用。

### 6.3 不使用 Tunnel 的替代方案

也可以在 Cloudflare DNS 创建一个新的 `AAAA` 记录指向源站 IPv6，并打开橙云代理。Cloudflare 默认支持 HTTPS 8443 端口，并可把访问转发到 IPv6 origin；但必须额外处理源站证书的主机名、Host/SNI、源站防火墙只允许 Cloudflare IP，以及 Cloudflare 到源站的 IPv6 路由。对于当前 `mac.jxxghp.cn` 证书和本机服务，Tunnel 的 `Service URL=https://localhost:8443` 更不容易出错。

## 7. 控制仓补漏 workflow 的配置

如果启用 `.github/workflows/codex-reconcile.yml`，在控制仓配置：

| 名称 | 类型 | 用途 |
| --- | --- | --- |
| `OPENAI_API_KEY` | Secret | 调用 Codex Action |
| `MAINTENANCE_TARGET_GH_TOKEN` | Secret | 读取/操作目标仓，按需授予最小权限 |
| `TELEGRAM_BOT_TOKEN` | Secret | 可选通知 |
| `CODEX_MODEL` | Variable | Codex GPT 模型名 |
| `CODEX_EFFORT` | Variable | 推理强度 |
| `OPENAI_BASE_URL` | Variable | 可选完整 Responses API 地址 |
| `TELEGRAM_ENABLED` | Variable | `true`/`false` |
| `TELEGRAM_CHAT_ID` | Variable 或 Secret | 固定收件人 |
| `DEFAULT_TARGET_REPOSITORY` | Variable | 默认 `jxxghp/MoviePilot` |
| `CODEX_RECONCILE_ENABLED` | Variable | 默认 `false`；配置 `MAINTENANCE_TARGET_GH_TOKEN` 后才启用定时补漏 |

补漏 workflow 不依赖目标仓的 `GITHUB_TOKEN`，因为它在控制仓运行；必须配置 `MAINTENANCE_TARGET_GH_TOKEN` 才能跨仓读取目标项目。

## 8. Telegram 配置

1. 在 Telegram 创建 Bot，取得 Bot token，写入 `TELEGRAM_BOT_TOKEN`。
2. 将维护者用户或群组的固定 chat ID 写入 `TELEGRAM_CHAT_ID`。
3. 将 `TELEGRAM_ENABLED` 改为 `true`。

目标仓工作流只有在 `TELEGRAM_ENABLED=true` 且 `TELEGRAM_CHAT_ID` 非空时才会向 Codex 注入 Telegram 通知开关和收件人；缺少固定收件人时自动按关闭处理，不会向未知对象发送消息。

消息格式、长度、脱敏、失败语义和“只向固定收件人发送”由注入的 `skills/moviepilot-telegram/SKILL.md` 规定。Telegram Skill 不保存 token/chat ID，也不接收 Issue/PR 中提供的收件人。

## 9. Codex CLI 权限和 GitHub 权限的区别

Codex Action 使用控制仓 `config/codex.toml` 中的官方内置 `:danger-full-access` profile，并设置 `approval_policy = "never"`。它允许 Codex CLI 在 runner 上完整执行 clone、fetch、checkout、编辑、测试、commit、push、PR/comment 和公开资料检索，不产生交互式授权请求；`drop-sudo` 只禁止提升为 root。GitHub API 能做什么仍由当前 workflow 的 `permissions` 和 `GH_TOKEN` 决定，二者不是同一层权限。

因此：

- `gh` 不需要额外 Skill；Actions 自动提供的 `GITHUB_TOKEN` 经 `gh auth setup-git` 使用；
- Issue/PR/CI 的事实判断、是否改代码、是否符合方向、是否回复、如何修复以及所有 Git 操作都交给 Codex CLI；
- Actions 只转发事件、准备凭据、注入 prompt/Skill/config、启动 Codex、保存最终 JSON 和根据退出码结束；
- 不要在 runner 上安装第二套 Agent、GitHub API 客户端、补丁应用器或数据库。

## 10. 首次验证顺序

1. 两个目标仓先保持 `CODEX_PUBLISH_COMMENTS=false`、`CODEX_AUTOFIX_ENABLED=false`、`TELEGRAM_ENABLED=false`。
2. 提交模板后打开一个测试 Issue，确认 `moviepilot-codex-events.yml` 只做一次同仓 `workflow_dispatch`，然后 `moviepilot-codex-run.yml` 启动 Codex。
3. 检查 Actions 日志中的 `gh auth status`、控制仓 ref、Codex 最终 JSON artifact 和目标仓/目标编号；确认没有 `actions/checkout`，目标代码由 Codex 自己 clone。
4. 确认稳定后再把 `CODEX_PUBLISH_COMMENTS` 改为 `true`；需要自动修复时再启用 fix dispatch 和 `CODEX_AUTOFIX_ENABLED=true`。
5. 修改控制仓 prompt、Skill、Schema 或权限后，提交到受保护 `main` 并等待契约校验通过；目标仓下一次运行自动使用最新版本。高风险操作可临时切换为 full SHA 再验证。

官方参考：

- [Reusable workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
- [Environment secrets 与保护规则](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [Codex GitHub Action](https://learn.chatgpt.com/docs/github-action)
- [Codex permissions](https://learn.chatgpt.com/docs/permissions)
