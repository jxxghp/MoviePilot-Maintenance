# 凭据与变量

所有值在 GitHub Environment 配置，不进入代码、prompt、Issue、PR、artifact 或 Codex 输出。`autofix` Environment 必须启用 required reviewers 和只允许目标 `v3` 的 workflow 来源；`propose` 也建议启用 required reviewers。

```text
OPENAI_API_KEY                # secret，openai/codex-action 的专用输入
MAINTENANCE_CONTROL_READ_TOKEN# secret，读取私有控制仓 prompt/schema；不传给 Codex
CODEX_MODEL                   # variable，Codex GPT 模型
CODEX_EFFORT                  # variable，Codex 推理强度
OPENAI_BASE_URL               # variable，可选模型代理地址
TELEGRAM_BOT_TOKEN            # secret，Codex 可选通知
TELEGRAM_CHAT_ID              # variable/secret，固定通知用户
TELEGRAM_ENABLED              # variable，默认 false；只控制是否把固定通知凭据注入 Codex
CODEX_PUBLISH_COMMENTS        # variable，默认 false
CODEX_AUTOFIX_ENABLED         # variable，默认 false
MAINTENANCE_CONTROL_REPOSITORY# variable，例如 jxxghp/MoviePilot-Maintenance
MAINTENANCE_CONTROL_REF       # variable，控制仓完整 commit SHA
MAINTENANCE_TARGET_GH_TOKEN   # secret，修复/控制仓补漏可选的最小权限 GitHub App/PAT token
```

GitHub Actions 自动提供的 `GITHUB_TOKEN` 只按 workflow 的 `permissions` 授权。shadow 分析使用只读权限，propose 仅增加 Issue comment 权限，修复 workflow 的写权限必须由 Environment approval 解锁。Codex permission profile 允许公网查资料，但禁止把 Issue/PR 内容当命令、执行不可信脚本或上传 Secrets；`drop-sudo` 和本地/私网保护仍保留。`MAINTENANCE_CONTROL_READ_TOKEN` 只用于 Actions 读取控制仓 prompt/schema，不传入 Codex；Codex 默认使用目标仓 `GITHUB_TOKEN`，修复/补漏可在受保护 Environment 中注入 `MAINTENANCE_TARGET_GH_TOKEN` 作为最小权限 GitHub App/PAT token。若需要自动触发下游 CI，优先使用后者；不要把它暴露给外部 PR 分析任务。不要使用公开仓的 `~/.codex/auth.json` 登录态作为 CI 凭据；不要把 `OPENAI_API_KEY` 或 `CODEX_API_KEY` 设置为整个 job 的环境变量。
