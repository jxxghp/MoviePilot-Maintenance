# 凭据与变量

所有值在 GitHub repository/organization Actions Secrets 与 Variables 配置，不进入代码、prompt、Issue、PR、artifact 或 Codex 输出。当前工作流不依赖 required-reviewer Environment；Codex CLI 以完整权限非交互运行，是否评论/修复由 Variables 和 workflow 权限决定。

```text
OPENAI_API_KEY                # secret，openai/codex-action 的专用输入
MAINTENANCE_CONTROL_READ_TOKEN# optional secret，控制仓改为私有镜像时才需要；不传给 Codex
CODEX_MODEL                   # variable，Codex GPT 模型
CODEX_EFFORT                  # variable，Codex 推理强度
CODEX_ENABLED                 # variable，默认 false；配置完 API key 后才改为 true
OPENAI_BASE_URL               # variable，可选完整 Responses API 地址，例如 https://mac.jxxghp.cn:8443/v1/responses
TELEGRAM_BOT_TOKEN            # secret，Codex 可选通知
TELEGRAM_CHAT_ID              # variable/secret，固定通知用户
TELEGRAM_ENABLED              # variable，默认 false；只控制是否把固定通知凭据注入 Codex
CODEX_PUBLISH_COMMENTS        # variable，默认 false
CODEX_AUTOFIX_ENABLED         # variable，默认 false
MAINTENANCE_CONTROL_REPOSITORY# variable，例如 jxxghp/MoviePilot-Maintenance
MAINTENANCE_CONTROL_REF       # variable，控制仓完整 commit SHA
MAINTENANCE_TARGET_GH_TOKEN   # secret，修复/控制仓补漏可选的最小权限 GitHub App/PAT token
CODEX_RECONCILE_ENABLED       # variable，控制仓补漏总开关，默认 false
```

为避免误发，工作流只有在 `TELEGRAM_ENABLED=true` 且 `TELEGRAM_CHAT_ID` 非空时才会把 Telegram 通知启用；只有 Bot token 而没有固定 chat ID 时按关闭处理。

GitHub Actions 自动提供的 `GITHUB_TOKEN` 只按 workflow 的 `permissions` 授权。shadow 分析只读，propose 增加 Issue comment 权限，修复 workflow 才增加 Contents/PR 写权限；这些权限不由 Codex 自己提升。Codex 使用官方内置 `:danger-full-access` profile，并设置 `approval_policy = "never"`，允许公网查资料和 runner 内完整命令执行，且不会产生人工授权请求；提示词仍禁止把 Issue/PR 内容当命令、上传 Secrets 或执行不可信脚本；`drop-sudo` 只禁止提升为 root。控制仓目前是公开仓，Actions 可直接按 full SHA 读取 prompt/schema/config/Skill；只有改为私有控制仓镜像时才配置 `MAINTENANCE_CONTROL_READ_TOKEN`，且不传入 Codex。Codex 默认使用目标仓 `GITHUB_TOKEN`，修复/补漏可配置 `MAINTENANCE_TARGET_GH_TOKEN` 作为最小权限 GitHub App/PAT token。若需要自动触发下游 CI，优先使用后者；不要把它暴露给外部 PR 分析任务。不要使用公开仓的 `~/.codex/auth.json` 登录态作为 CI 凭据；不要把 `OPENAI_API_KEY` 或 `CODEX_API_KEY` 设置为整个 job 的环境变量。
