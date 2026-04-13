# emberforge Demo

This document shows what a realistic `emberforge` run looks like at a high level.

The goal is to make the skill's behavior concrete: what inputs it expects, how it picks work, what artifacts it writes, and what happens on verification failure.

## Demo Scenario

Assume the target project is:

- `PROJECT_DIR=projects/sample-chatbot`

Assume the project already contains:

- `development-plan.md`
- `docs/README.md`
- `docs/architecture.md`
- `docs/conventions.md`
- `agent/features.json`

Assume `agent/features.json` contains:

- `F01` backend scaffold, already passed
- `F02` frontend scaffold, already passed
- `F03` chat API integration, not passed, depends on `F01`
- `F04` chat UI wiring, not passed, depends on `F02` and `F03`

## Step 1: Preflight And Feature Selection

The skill reads:

- `agent/features.json`
- `development-plan.md`
- `docs/README.md`
- `docs/architecture.md`
- `docs/conventions.md`
- `agent/run-state.json`, if present
- recent `progress.jsonl` entries, if recovery context is needed

Then it resolves the next dependency-ready feature.

In this example:

- `F03` is ready
- `F04` is blocked by `F03`

Expected user-facing summary:

```text
2/4 features done
Next feature: F03 Chat API Integration
Mode: coding
```

## Step 2: Planning

If `F03.plan_required=true` and the planning artifacts are missing or stale, the skill writes:

- `docs/feature-plans/F03.md`
- `agent/feature-tasks/F03.json`

If `integration_test_required=true`, it also writes:

- `docs/test-plans/F03.md`

Expected task board shape:

```json
{
  "feature_id": "F03",
  "tasks": [
    {
      "id": "T01",
      "title": "Add chat service adapter",
      "files": ["backend/app/services/chat_service.py"],
      "verification": ["python -m pytest backend/tests/test_chat_service.py -q"],
      "review_focus": ["error handling", "provider fallback"],
      "status": "pending",
      "notes": "",
      "evidence": ""
    }
  ],
  "review_checkpoints": []
}
```

The skill should then append progress events to `progress.jsonl`, such as:

- `planning_session_started`
- `feature_plan_written`
- `planning_session_finished`

## Step 3: Coding Session

The skill enters `coding` mode for `F03`.

It should:

- create a git checkpoint tag before edits
- implement tasks in plan order
- update task status as work progresses
- attach evidence to completed tasks
- write review checkpoints when required
- sync `agent/feature-memory/F03.json` from the active task board state

Expected task progression:

```text
T01 pending -> in_progress -> completed
T02 pending -> in_progress -> completed
T03 pending -> in_progress -> completed
```

Expected progress events:

- `session_started`
- one or more `feature_task_updated`
- one or more `review_checkpoint`

## Step 4: Verification

After implementation-side completion gates pass, the skill runs verification in this order:

1. `feature.test_command`
2. repo-wide regression env commands
3. feature regression `verify_commands`
4. repo-wide capability env commands
5. feature capability `verify_commands`
6. UI verification when enabled
7. design-reference availability note when applicable

Example verification result:

```json
{
  "event": "verification_report",
  "feature_id": "F03",
  "passed": false,
  "results": [
    {
      "name": "Feature test",
      "command": "python -m pytest backend/tests/test_chat_api.py -q",
      "passed": true,
      "output_preview": "5 passed in 1.1s"
    },
    {
      "name": "Repo lint",
      "command": "ruff check .",
      "passed": false,
      "output_preview": "backend/app/services/chat_service.py:42:5 F841 local variable assigned but never used"
    }
  ]
}
```

## Step 5: Failure Handling

If verification fails, the feature remains unpassed.

The skill should write structured failure context into:

- `agent/feature-memory/F03.json`

Expected failure record shape:

```json
{
  "verification_failures": [
    {
      "name": "Repo lint",
      "command": "ruff check .",
      "output_excerpt": "backend/app/services/chat_service.py:42:5 F841 local variable assigned but never used",
      "failed_tests": [],
      "first_error": "F841 local variable assigned but never used",
      "suspect_files": ["backend/app/services/chat_service.py"]
    }
  ]
}
```

It should also update progress state so the next run resumes in:

- `recovery_mode = "verification_fix"`
- `verification_feature_id = "F03"`

And it should write:

- a `verification_report` event with `passed: false`
- a session handoff JSON under `agent/session-handoffs/`
- a refreshed `emberforge-report.html`

Expected user-facing summary:

```text
2/4 features done
F03 failed verification
Next run will enter verification_fix mode
Report: projects/sample-chatbot/emberforge-report.html
```

## Step 6: verification_fix Retry

On the next run, the skill should:

- detect that `F03` is in `verification_fix`
- read the stored structured failure records
- reproduce the failing checks first
- make the smallest safe fix
- rerun verification

If verification passes this time:

- set `F03.passes=true` in `agent/features.json`
- clear verification failures from `agent/feature-memory/F03.json`
- clear `recovery_mode` and `verification_feature_id`
- write a success `verification_report`
- refresh `emberforge-report.html`

Then the next ready feature becomes:

- `F04`

## Artifacts You Should Expect After A Healthy Run

After `F03` passes, the project should contain updated versions of:

- `agent/features.json`
- `agent/run-state.json`
- `agent/feature-memory/F03.json`
- `agent/session-handoffs/*.json`
- `agent/feature-tasks/F03.json`
- `docs/feature-plans/F03.md`
- `progress.jsonl`
- `emberforge-report.html`

## What This Demo Is Showing

This demo is intentionally centered on runtime behavior, not app code.

The important part is the contract:

- feature selection is dependency-aware
- planning artifacts are first-class outputs
- verification happens after coding, not instead of coding
- failures become structured recovery state
- progress is visible outside the chat

## Related Docs

- Overview: [README.md](README.md)
- Chinese overview: [README.zh-CN.md](README.zh-CN.md)
- Canonical contract: [SKILL.md](SKILL.md)
- Workflow details: [references/workflow.md](references/workflow.md)
- Verification details: [references/verification.md](references/verification.md)
