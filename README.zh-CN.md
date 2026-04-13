# emberforge

[English](README.md) | 中文

用 AI coding agent 做开发时，emberforge 提供一套可恢复、可审计、可按依赖推进的标准交付流程。

如果这个项目帮你减少返工、提升交付稳定性，欢迎给一个 Star 支持。

## 为什么用 emberforge

很多 AI 开发会话在任务变长后会出现状态漂移、验证跳过、失败难恢复。

emberforge 把规划、编码、验证、恢复、报告统一到一个明确契约里。

## 改造前 vs 改造后

| 没有结构化流程 | 使用 emberforge |
|---|---|
| 进度只在聊天里 | 进度写入 `progress.jsonl` |
| 验证步骤随意 | 验证顺序固定且可复现 |
| 失败后很难续跑 | `verification_fix` 有结构化恢复上下文 |
| 容易越过依赖顺序 | 按 dependency-ready 自动调度 |
| 状态汇报靠手工 | 自动刷新 `emberforge-report.html` |

![emberforge 报告示例](images/sample-report.png)

## 60 秒快速上手

1. 准备目标项目的基础文件。
2. 在你的 coding agent 中加载本 skill。
3. 发送一句触发词：Help me build an xxx project.

目标项目至少包含：

- `{PROJECT_DIR}/development-plan.md`
- `{PROJECT_DIR}/docs/README.md`
- `{PROJECT_DIR}/docs/architecture.md`
- `{PROJECT_DIR}/agent/features.json`

可参考最小示例结构：[sample-project/](sample-project)

健康运行后常见产物：

- `agent/run-state.json`
- `agent/feature-memory/Fxx.json`
- `agent/session-handoffs/`
- `progress.jsonl`
- `emberforge-report.html`

完整流程演示见 [demo.md](demo.md)

## 支持的 Agent

只要支持自定义 skill 或系统提示词的 AI coding agent，通常都可以接入。

| Agent | 使用方式 |
|---|---|
| Claude Code | 将 skill 放到项目里后，发送 Help me build xxx |
| Codex | 加载为 skill 后，发送 Help me build xxx |
| OpenCode | 加载为 skill 后，发送 Help me build xxx |
| Qclaw | 加载为 skill 后，发送 Help me build xxx |
| OpenClaw | 加载为 skill 后，发送 Help me build xxx |
| Cursor / Windsurf / Copilot | 写入 .instructions.md 或同类文件后发送 Help me build xxx |

## 这个 Skill 能做什么

- 把 `agent/features.json` 当作执行 backlog
- 为缺失计划的 feature 生成 plan 和 task board
- 一次实现一个依赖已满足的 feature
- 按真实顺序跑验证流水线
- 在验证失败时写入 `verification_fix` 所需的恢复上下文
- 持续往 `progress.jsonl` 追加事件
- 每次验证后刷新 `emberforge-report.html`

它不是一个泛用编码提示词，而是一个带明确运行契约的 skill。

## 和普通提示词流程的区别

| 维度 | 普通提示词流程 | emberforge |
|---|---|---|
| 状态模型 | 主要在会话内 | 以文件产物为中心 |
| 失败恢复 | 依赖人工回忆 | `verification_fix` 结构化恢复 |
| 完成标准 | 主观判断 | completion gate + verification loop |
| 任务调度 | 常常隐式 | 依赖就绪后再执行 |
| 汇报能力 | 零散笔记 | 事件日志 + HTML 报告 |

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

如果 emberforge 对你有帮助，欢迎 Star。你的支持会直接推动模板、验证契约和文档持续迭代。
