# emberforge

`emberforge` is an IDE skill for structured, recoverable software delivery.

It is useful when you want planning, coding, verification, recovery, and reporting to follow one explicit contract inside an editor session.

## Supported Agents

emberforge works with any AI coding agent that supports custom skills or system prompts:

| Agent | How to use |
|-------|-----------|
| **Claude Code** | Place skill in your project, then: *"Help me build xxx"* |
| **Codex** | Load as a skill, then: *"Help me build xxx"* |
| **OpenCode** | Load as a skill, then: *"Help me build xxx"* |
| **Qclaw** | Load as a skill, then: *"Help me build xxx"* |
| **OpenClaw** | Load as a skill, then: *"Help me build xxx"* |
| **Cursor / Windsurf / Copilot** | Add to `.instructions.md` or equivalent, then: *"Help me build xxx"* |

> **Quick start:** prepare your project files (`development-plan.md`, `features.json`, etc.), load the skill, and just say:
>
> *"Help me build an xxx project"*
>
> emberforge takes it from there: planning, coding, verification, and reporting, all in one structured flow.

## What This Skill Does

- reads `agent/features.json` as the execution backlog
- generates missing feature plans and task boards
- implements one dependency-ready feature at a time
- runs the real verification sequence
- persists failure context for `verification_fix` retries
- appends progress events to `progress.jsonl`
- regenerates `emberforge-report.html`

This is not a generic coding prompt. It is a skill with a concrete runtime contract.

## Why It Exists

Most coding-agent sessions fail on longer work because they lose state, skip verification, and recover poorly after failed runs.

`emberforge` addresses that with one opinionated workflow:

- planning before coding
- task-by-task implementation
- explicit completion gates
- structured verification
- durable recovery artifacts
- audit-friendly event logs

## Included Files

- [SKILL.md](SKILL.md): canonical operating contract for the skill
- [README.zh-CN.md](README.zh-CN.md): Chinese overview for this skill
- [demo.md](demo.md): end-to-end walkthrough with expected artifacts
- [agents/](agents): planner, coder, and reviewer role prompts
- [references/workflow.md](references/workflow.md): lifecycle and scheduling rules
- [references/feature-schema.md](references/feature-schema.md): `features.json` and task-board contract
- [references/verification.md](references/verification.md): verification order and failure handling
- [references/reporting.md](references/reporting.md): event log and HTML report contract
- [references/gotcha-library.md](references/gotcha-library.md): implementation gotchas used during delivery

## Required Project Shape

The target project is expected to already contain:

- `{PROJECT_DIR}/development-plan.md`
- `{PROJECT_DIR}/docs/README.md`
- `{PROJECT_DIR}/docs/architecture.md`
- `{PROJECT_DIR}/agent/features.json`

Strongly recommended:

- `{PROJECT_DIR}/docs/conventions.md`

## Runtime Artifacts This Skill Uses

The skill does not invent a private `.skill-state.json`.

It uses these runtime artifacts:

- `agent/features.json`
- `agent/run-state.json`
- `agent/feature-memory/Fxx.json`
- `agent/session-handoffs/`
- `progress.jsonl`
- `docs/project-lessons.md`
- `emberforge-report.html`

`features.json` remains the source of truth for completion via each feature's `passes` field.

## How The Flow Works

1. Resolve the target project and run the two-phase initializer when required.
2. Pick the next dependency-ready feature from `agent/features.json`.
3. Generate planning artifacts if the feature requires them.
4. Implement the feature task by task.
5. Enforce completion gates before verification.
6. Run the real verification sequence.
7. On failure, persist structured recovery context and re-enter `verification_fix`.
8. On success, mark the feature passed and refresh `emberforge-report.html`.

## What Makes This Skill Different

`emberforge` is opinionated about delivery:

- it uses the real runtime contract, not a simplified blog-post version
- it treats verification failures as first-class state
- it writes structured progress events instead of leaving state in chat history
- it keeps project-relative paths scoped to `PROJECT_DIR`

## Source Of Truth

For this skill, the source of truth is this directory:

1. [SKILL.md](SKILL.md)
2. [references/](references)
3. [agents/](agents)

If these files drift, update them together so the contract stays internally consistent.

## Non-Goals

- acting like an unstructured general-purpose coding prompt
- inventing a parallel state format
- bypassing verification before marking a feature complete
- using external coding agents or remote code-generation services during a run

## For Maintainers

Before changing this skill, verify that these files still agree with each other:

- `SKILL.md`
- `README.md`
- `README.zh-CN.md`
- `demo.md`
- `agents/*`
- `references/*`

Contribution and disclosure guidelines live in:

- [CONTRIBUTING.md](CONTRIBUTING.md)
- [SECURITY.md](SECURITY.md)
- [LICENSE](LICENSE)

Additional docs:

- Chinese README: [README.zh-CN.md](README.zh-CN.md)
- Demo walkthrough: [demo.md](demo.md)
