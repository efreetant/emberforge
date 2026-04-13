# Feature Schema Reference

This file describes the `agent/features.json` contract and the planning/task-board artifacts that this skill expects.

## Top-Level JSON

```json
{
  "project": "project-name",
  "description": "Project summary",
  "features": []
}
```

## Feature Fields

### Identity

- `id`
- `phase`
- `name`
- `description`
- `acceptance_criteria`

### Scheduling

- `depends_on`
- `passes`

### Verification

- `test_command`
- `verify_commands`
- `ui_verify`: `off | smoke | module`
- `ui_verify_target`
- `integration_test_required`
- `integration_test_plan_path`

`verify_commands` entries use:

```json
{
  "name": "Human-readable name",
  "command": "shell command",
  "suite": "regression",
  "required": true
}
```

Rules:
- regression checks block by default
- capability checks are optional by default
- if `VERIFY_CAPABILITY_REQUIRED=true`, capability checks block unless `required:false`

### Planning / context

- `plan_required`
- `plan_path`
- `task_board_path`
- `docs_refs`
- `user_story_refs`
- `expected_files`
- `tech_tags`
- `review_required`

## Task Board Schema

The current emberforge task board is compatible with `write_feature_plan`, `update_feature_task`, and `mark_feature_complete`.

```json
{
  "feature_id": "F03",
  "feature_name": "Login Module",
  "summary": "Short planning summary",
  "plan_path": "docs/feature-plans/F03.md",
  "task_board_path": "agent/feature-tasks/F03.json",
  "integration_test_plan_path": "docs/test-plans/F03.md",
  "user_story_refs": ["docs/ux/login.md"],
  "tasks": [
    {
      "id": "T01",
      "title": "Add auth route",
      "description": "Implement the route and tests",
      "files": ["backend/app/routes/auth.py"],
      "verification": ["pytest backend/tests/test_auth.py -v"],
      "review_focus": ["error handling", "response shape"],
      "status": "pending",
      "notes": "",
      "evidence": ""
    }
  ],
  "review_checkpoints": []
}
```

Important:
- only `completed` counts as done
- completed tasks should include evidence
- review checkpoints are separate records, not fake tasks

## Review Checkpoint Shape

```json
{
  "id": "RC-01",
  "task_id": "T03",
  "stage": "pre-complete",
  "outcome": "pass",
  "notes": "Reviewed API contract and edge cases.",
  "summary": "Reviewed API contract and edge cases."
}
```

## Durable Runtime Artifacts

Do not model recovery around `.skill-state.json`. The current emberforge uses:
- `agent/run-state.json`
- `agent/feature-memory/Fxx.json`
- `agent/session-handoffs/*.json`
- `progress.jsonl`
- `emberforge-report.html`

`features.json` still remains the completion source of truth via the `passes` field.

`emberforge-report.html` is regenerated after each feature verification (pass or fail) and at project completion. See `references/reporting.md` for the HTML template and event schema.

## Relevant Env Toggles

- `VERIFY_LINT_CMD`
- `VERIFY_TYPE_CMD`
- `VERIFY_BUILD_CMD`
- `VERIFY_REGRESSION_CMD_*`
- `VERIFY_CAPABILITY_CMD_*`
- `VERIFY_CAPABILITY_REQUIRED`
- `UI_DEBUG_ENABLED`
- `UI_DEBUG_MODE`
- `UI_DEBUG_BASE_URL`
- `UI_DEBUG_SERVER_CMD`
- `UI_DEBUG_CHECK_CMD_*`
- `DESIGN_ENABLED`
- `DESIGN_PROVIDER`
- `DESIGN_MODEL`
