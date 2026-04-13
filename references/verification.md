# Verification Reference

This file defines the verification behavior for this skill.

## Completion Gates Before Verification

Do not verify until:
- the plan exists when `plan_required=true`
- the integration test plan exists when `integration_test_required=true`
- the task board exists
- all tasks are `completed`
- review checkpoints exist when `review_required=true`

Also remember:
- `mark_feature_complete` rejects completed tasks that lack evidence

## Verification Order

### 1. Primary feature test

Always run:
- `feature.test_command`

### 2. Repo-wide regression checks from `.env`

If set, run:
- `VERIFY_LINT_CMD`
- `VERIFY_TYPE_CMD`
- `VERIFY_BUILD_CMD`
- `VERIFY_REGRESSION_CMD_*`

Repo-wide frontend commands may be skipped until a frontend scaffold exists.

### 3. Feature regression `verify_commands`

Feature `verify_commands` with `suite` omitted or `suite="regression"` block by default unless `required:false`.

### 4. Repo-wide capability checks from `.env`

If set, run:
- `VERIFY_CAPABILITY_CMD_*`

### 5. Feature capability `verify_commands`

Feature `verify_commands` with `suite="capability"`:
- optional by default
- become blocking when `VERIFY_CAPABILITY_REQUIRED=true`
- remain non-blocking when explicitly marked `required:false`

### 6. UI verification

Run only when:
- `UI_DEBUG_ENABLED=true`
- `feature.ui_verify` is `smoke` or `module`

Use `feature.ui_verify_target` when present; otherwise fall back to `UI_DEBUG_BASE_URL`.

UI checks include:
- page health
- screenshot capture
- console/network errors
- `UI_DEBUG_CHECK_CMD_*`

These are capability-suite checks.

### 7. Design-reference availability note

When:
- `DESIGN_ENABLED=true`
- image refs exist in `docs_refs`
- a screenshot was captured

Record that manual visual comparison is available. This is informational only.

## Verdict

- any failing required regression/capability check => feature fails verification
- optional capability failures are recorded but do not block

## Failure Handling

On verification failure:
- keep `passes=false`
- **write structured failure records to `agent/feature-memory/{feature_id}.json`**: for each failed check, store `name`, `command`, `output_excerpt` (up to 1500 chars), `failed_tests` (names parsed from output), `first_error`, `suspect_files`, `repro_steps`
- set `recovery_mode = "verification_fix"` and `verification_feature_id` in progress state
- preserve the raw verification result dicts in `progress.verification_artifacts` (used to format context for the next session)
- write a session handoff with `result: "verification_failed"`
- ensure the next run enters `verification_fix`
- feed the latest failed commands and output back into the coder context

## Success Handling

On verification success:
- mark the feature passed in `agent/features.json`
- **clear all verification failure records** from `agent/feature-memory/{feature_id}.json`
- clear `recovery_mode`, `verification_feature_id`, and `verification_artifacts` in progress state
- record the last clean git commit hash
- write a session handoff with `result: "completed"`
- continue to the next ready feature unless the user requested one-feature mode
