# 发布与试点

1. 使用公开的 `jxxghp/MoviePilot-Maintenance` 控制仓，记录完整 control ref SHA。
2. 在目标仓添加事件 intake 和对应的 `*-codex-run.yml`，按 `docs/github-configuration.md` 配置 repository/organization Actions Secrets/Variables；先保持 `CODEX_ENABLED=false`，intake 只用带 `actions: write` 的 `GITHUB_TOKEN` dispatch trusted run，不使用 `actions/checkout`，也不接触 OpenAI/Telegram Secret。
3. 配置 `OPENAI_API_KEY` 等必需项后将 `CODEX_ENABLED=true`，先只触发 Codex 分析，不允许评论；回放至少 30 个有已知结论的历史 Issue/PR，检查 Codex 最终输出、工具调用和提示词注入样例。
4. 开启 `CODEX_PUBLISH_COMMENTS=true` 后，事件桥才调用有 Issue comment 权限的 `codex-propose.yml`；由 Codex 自己通过 `gh issue comment` 回复，运行中不等待人工批准。
5. 配置 fix dispatch 所需的写权限和 `CODEX_AUTOFIX_ENABLED=true`；Codex 自己 clone、修复、测试、commit、push 和建 PR，运行中不等待人工批准。
6. 修复试点稳定后，再由维护者决定是否允许更多类别；永远不让 Codex 直接推 `v3` 或自动 merge，除非新增明确授权和平台保护。
7. 前端发布、后端发布、数据库/安全/依赖/公共契约/架构文件仍然由控制仓 prompt、Actions Variables 和分支保护预先设定边界；Codex 运行中不等待人工门禁。
