# emberforge

`emberforge` 是一个面向结构化、可恢复交付流程的 IDE skill。

如果你想在 IDE 会话里用一套明确的规划、编码、验证、恢复、报告契约来推进项目，这个 skill 就是给这个场景准备的。

## 这个 Skill 能做什么

- 把 `agent/features.json` 当作执行 backlog
- 为缺失计划的 feature 生成 plan 和 task board
- 一次实现一个依赖已满足的 feature
- 按真实顺序跑验证流水线
- 在验证失败时写入 `verification_fix` 所需的恢复上下文
- 持续往 `progress.jsonl` 追加事件
- 每次验证后刷新 `emberforge-report.html`

它不是一个“泛用编码提示词”，而是一个带明确运行契约的 skill。

## 它解决什么问题

普通 coding agent 做长任务时，常见问题是：

- 会话一长就丢上下文
- 验证不严格，甚至直接跳过
- 跑挂以后没有结构化恢复信息
- 过程只留在聊天记录里，不可审计

`emberforge` 通过一套明确的交付模型来解决这些问题：

- 先规划，再编码
- 按 task 推进，不是一口气乱改
- 完成前必须过 completion gates
- 验证失败会留下结构化失败上下文
- 进度、报告、handoff 都落盘

## 目录里有什么

- [SKILL.md](SKILL.md)：skill 的正式契约
- [README.md](README.md)：英文版概览
- [demo.md](demo.md)：一次完整 run 的演示说明
- [agents/](agents)：planner、coder、reviewer 角色提示
- [references/workflow.md](references/workflow.md)：生命周期与调度规则
- [references/feature-schema.md](references/feature-schema.md)：`features.json` 与 task board 结构
- [references/verification.md](references/verification.md)：验证顺序与失败处理
- [references/reporting.md](references/reporting.md)：事件日志与 HTML 报告契约
- [references/gotcha-library.md](references/gotcha-library.md)：实现中的 gotcha 参考

## 目标项目需要满足什么结构

至少需要：

- `{PROJECT_DIR}/development-plan.md`
- `{PROJECT_DIR}/docs/README.md`
- `{PROJECT_DIR}/docs/architecture.md`
- `{PROJECT_DIR}/agent/features.json`

强烈建议再有：

- `{PROJECT_DIR}/docs/conventions.md`

## 这个 Skill 使用哪些运行态文件

它不会发明私有的 `.skill-state.json`。

它使用的是这套标准运行态 artifacts：

- `agent/features.json`
- `agent/run-state.json`
- `agent/feature-memory/Fxx.json`
- `agent/session-handoffs/`
- `progress.jsonl`
- `docs/project-lessons.md`
- `emberforge-report.html`

其中 `features.json` 里的 `passes` 仍然是功能是否完成的最终真相。

## 工作流程

1. 先确认目标项目，并在需要时运行两阶段 initializer。
2. 从 `agent/features.json` 里选出下一个依赖就绪的 feature。
3. 如果该 feature 要计划且计划缺失，就先生成 planning artifacts。
4. 按 task board 逐项实现。
5. 完成后先检查 completion gates，再进入验证。
6. 按真实验证顺序运行检查。
7. 如果失败，写入恢复上下文并进入 `verification_fix`。
8. 如果成功，标记 feature 通过并刷新 `emberforge-report.html`。

## 它和普通 Skill 的区别

`emberforge` 很“死板”，这是刻意设计的：

- 它追随真实 runtime，而不是一版被简化过的说明
- 它把验证失败当成正式状态，而不是一句“下次再修”
- 它把进度写进结构化文件，而不是只留在聊天里
- 它要求所有项目路径都相对 `PROJECT_DIR`

## 事实来源

对这个 skill 来说，事实来源就是它自己的目录：

1. [SKILL.md](SKILL.md)
2. [references/](references)
3. [agents/](agents)

如果这些文件彼此不一致，应当一起修正，保持契约自洽。

## 非目标

- 退化成一个无结构的通用编码提示词
- 发明另一套平行状态格式
- 在没过验证时把 feature 标记为完成
- 在运行过程中委托外部 coding agent 或远程代码生成服务

## 维护者入口

改这个 skill 之前，至少检查这些文件是否仍一致：

- `SKILL.md`
- `README.md`
- `README.zh-CN.md`
- `demo.md`
- `agents/*`
- `references/*`

附加文档：

- 贡献指南：[CONTRIBUTING.md](CONTRIBUTING.md)
- 安全策略：[SECURITY.md](SECURITY.md)
- 许可证：[LICENSE](LICENSE)
