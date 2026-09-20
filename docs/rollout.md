# 发布与试点

1. 创建私有 `MoviePilot-Maintenance` 控制仓，提交本目录后记录完整 control ref SHA。
2. 在目标仓添加事件 intake 和对应的 `*-codex-run.yml`，配置 `shadow` Environment、`OPENAI_API_KEY`、控制仓读取 token、模型变量；intake 只用带 `actions: write` 的 `GITHUB_TOKEN` dispatch trusted run，不使用 `actions/checkout`，也不接触 OpenAI/Telegram Secret。
3. 先只触发 Codex 分析，不允许评论；回放至少 30 个有已知结论的历史 Issue/PR，检查 Codex 最终输出、工具调用和提示词注入样例。
4. 创建并保护 `propose` Environment，开启 `CODEX_PUBLISH_COMMENTS=true` 后，事件桥才调用有 Issue comment 权限的 `codex-propose.yml`；由 Codex 自己通过 `gh issue comment` 回复，维护者检查中文报告和证据。
5. 单独创建 `autofix` Environment，启用 required reviewer 和 `CODEX_AUTOFIX_ENABLED=true`；先手工选择低风险 Issue，让 Codex 自己 clone、修复、测试、commit、push 和建 PR。
6. 修复试点稳定后，再由维护者决定是否允许更多类别；永远不让 Codex 直接推 `v3` 或自动 merge，除非新增明确授权和平台保护。
7. 前端发布、后端发布、数据库/安全/依赖/公共契约/架构文件仍然由维护者在 Codex prompt 和 Environment 中保持人工门禁。
