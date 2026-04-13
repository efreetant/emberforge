---
name: emberforge
description: Execute the emberforge workflow on a prepared project: read agent/features.json, create or refresh feature plans, implement one dependency-ready feature, run the verification loop, persist recovery context in the standard skill artifacts, and continue until completion. Use when the user wants structured, recoverable feature-by-feature delivery inside the skill.
---

# emberforge Skill

You are acting as the `emberforge` runtime.

Do not follow stale copies, old notes, or external blog posts when they conflict with this skill package. The source of truth is:
1. `SKILL.md`
2. the files under `references/`
3. the files under `agents/`
4. `README.md`

Read these first:
- `references/workflow.md`
- `references/feature-schema.md`
- `references/verification.md`
- `references/reporting.md`

## Required Project Shape

The target project must already contain:
- `{PROJECT_DIR}/development-plan.md`
- `{PROJECT_DIR}/docs/README.md`
- `{PROJECT_DIR}/docs/architecture.md`
- `{PROJECT_DIR}/agent/features.json`

`docs/conventions.md` is strongly recommended.

## Runtime Contract You Must Mirror

### State and recovery artifacts

Use the standard skill artifacts, not a private `.skill-state.json` file:
- `agent/features.json`
- `agent/run-state.json`
- `agent/feature-memory/Fxx.json`
- `agent/session-handoffs/`
- `progress.jsonl`
- `docs/project-lessons.md`
- `emberforge-report.html`

`features.json` is still the source of truth for completion: only `passes` is flipped after successful verification.

### Recording contract

Append structured JSON lines to `{PROJECT_DIR}/progress.jsonl` using the event schema in `references/reporting.md`.

The skill must record events at these points:
- `session_started` / `planning_session_started` — before starting work
- `feature_plan_written` — after writing plan + task board
- `feature_task_updated` — after each task status change
- `review_checkpoint` — after recording a review checkpoint
- `verification_report` — after running the verification sequence
- `session_finished` / `planning_session_finished` — after completing work on a feature

Additionally, write a session handoff JSON to `agent/session-handoffs/` at the end of each feature session (pass or fail).

### Env-driven toggles

Runtime behavior comes from `.env`, not from a skill-local config file:
- `UI_DEBUG_ENABLED`
- `UI_DEBUG_MODE`
- `UI_DEBUG_BASE_URL`
- `UI_DEBUG_SERVER_CMD`
- `UI_DEBUG_CHECK_CMD_*`
- `DESIGN_ENABLED`
- `DESIGN_PROVIDER`
- `DESIGN_MODEL`
- `VERIFY_LINT_CMD`
- `VERIFY_TYPE_CMD`
- `VERIFY_BUILD_CMD`
- `VERIFY_REGRESSION_CMD_*`
- `VERIFY_CAPABILITY_CMD_*`
- `VERIFY_CAPABILITY_REQUIRED`

Capability checks are non-blocking by default unless `VERIFY_CAPABILITY_REQUIRED=true`.

### Template auto-loading system

The skill uses a template library at `docs/templates/template-*.md`. Each template has YAML frontmatter:

```yaml
---
name: template-vuetify-complete
tech_tags: ["vue3", "vuetify", "vite", "pinia", "vue-router"]
applies_to: ["frontend-scaffold", "frontend"]
description: Vue 3 + Vuetify 3 complete setup guide
---
```

During Phase 2 of initialization, **before the agent session starts**, template matching and copy should run automatically:

1. Collect all `tech_tags` from every feature in `agent/features.json`
2. Scan `docs/templates/` for `template-*.md` files whose `tech_tags` intersect with the project's tags
3. Copy matched templates to the project's `docs/` directory
4. Update each feature's `docs_refs` array — if a feature's `tech_tags` overlap with a template's tags, that template path (`docs/template-*.md`) is appended

Available templates:
| Template | tech_tags | applies_to |
|---|---|---|
| `template-fastapi-complete` | fastapi, uvicorn, sqlalchemy, pydantic, pytest | backend-scaffold, backend |
| `template-pytest-patterns` | pytest, httpx, pytest-cov, pytest-asyncio | backend-scaffold, backend |
| `template-vue-router-modern` | vue-router, vue3 | frontend-scaffold, frontend |
| `template-vuetify-complete` | vue3, vuetify, vite, pinia, vue-router, vite-plugin-vuetify | frontend-scaffold, frontend |

The skill must follow this contract: during Phase 2, match templates by tag intersection, copy them, and wire `docs_refs`. The Phase 2 pass then **verifies** the result rather than doing the matching itself.

## Invocation Guardrails (Do Not Skip)

Any user request that implies software delivery must enter the full emberforge lifecycle, even when phrased casually.

Treat all of the following as full workflow requests:
- "help me dev xxx"
- "help me build xxx"
- "implement xxx"
- "fix xxx"

For those requests, do this in order and do not jump directly to coding:
1. Step 0 (resolve target project and mode)
2. Step 0.5 (two-phase initializer when required)
3. Step 1 preflight (including runtime artifact reads)
4. Step 2 feature selection
5. Step 3 planning when required or when planning fields are missing/invalid
6. Step 4+ execution and verification

Planning fallback rules:
- if `plan_required` is missing on a feature, treat it as `true`
- if `plan_path` or `task_board_path` is missing/invalid, run planner and repair those fields before coding

## Step 0 — Resolve the Target Project

Extract or confirm:
- `PROJECT_DIR`
- whether the user wants one feature only or full loop mode
- whether the user wants final reviewer audit after all features pass

When the user mentions UI/debug/design behavior, map that intent to the real `.env` knobs above. Do not invent `DESIGN_REF` or `.skill-config` as the canonical runtime source.

## Step 0.5 — Two-Phase Initializer

Run the two-phase initializer defined here **once per project** before any feature work begins. If `agent/run-state.json` does not exist or `git_initialized` is false, you must run the initializer before proceeding to Step 1.

### Phase 1: Environment Setup (fast, critical checkpoint)

Goal: verify toolchain and create a clean git starting point. Do **nothing** else.

1. **Verify environment** — run `python --version`, `node --version`, `git status`; init git if needed
2. **Quick project scan** — confirm `agent/features.json` is valid JSON and `development-plan.md` exists
3. **Create virtualenv** — `uv venv` at project root; do NOT install packages
4. **Initial git commit** — `git add development-plan.md .env.example .gitignore agent/ && git commit -m "chore(emberforge): initial project scaffold"`
5. **Mark Phase 1 complete** — call `mark_feature_complete` with `feature_id="INIT_PHASE1"`

Strict rules for Phase 1:
- Do NOT install dependencies (no pip/npm/uv add)
- Do NOT create `backend/` or `frontend/` or any application code
- Do NOT modify `features.json` or `docs/`

Record `session_started` (feature_id=`INIT_PHASE1`) before work and `session_finished` after.

If Phase 1 fails, stop and report: "Re-run init after fixing the Phase 1 issue."

### Phase 2: System Configuration (moderate complexity)

Goal: auto-load templates, validate `features.json`, populate `docs/`. Phase 1 is already done.

1. **Verify auto-loaded templates** — the emberforge copies `template-*.md` files to `docs/` based on `tech_tags` before this phase runs. Confirm which templates landed in `docs/` and that `docs_refs` in `features.json` reference them. Fix any missing links.
2. **Validate and fix `agent/features.json`** — apply two audits:
   - **Path rules** (fix silently): bare executable names only (no `.venv/Scripts/` prefix); backend commands from project root (no `cd backend &&`); frontend commands use `--prefix frontend`
   - **Verify loop audit** — classify each feature as `backend-scaffold`, `backend`, or `frontend` from `test_command`/`tech_tags`, then add missing verification commands per the checklist:
     - *backend-scaffold*: import smoke, ruff check, ruff format
     - *backend* (phase 2+): pytest all, ruff check, ruff format, coverage ≥60%, frontend build (if frontend exists), alembic check (if alembic in tech_tags)
     - *frontend*: backend smoke, ruff check, ruff format
   - Never duplicate `test_command` in `verify_commands`; never add frontend build before frontend scaffold exists
3. **Populate `docs/`** — update `docs/architecture.md`, `docs/conventions.md`, `docs/README.md` with actual tech stack and project-specific rules
4. **Final git commit** — `git add agent/features.json docs/ && git commit -m "chore(emberforge): configure system and auto-load templates"`
5. **Mark INIT complete** — call `mark_feature_complete` with `feature_id="INIT"`

Strict rules for Phase 2:
- Do NOT install dependencies
- Do NOT create application code or scaffold directories
- Do NOT implement features

Record `session_started` (feature_id=`INIT_PHASE2`) before work and `session_finished` (feature_id=`INIT`) after both phases complete.

### After initialization

Set `git_initialized = true` in progress state. The project is now ready for Step 1 → Step 6 feature loop.

## Step 1 — Preflight

Before picking a feature:
- confirm the required files exist
- read `agent/features.json`
- read `docs/README.md`, `docs/architecture.md`, and `development-plan.md`
- if present, read `docs/conventions.md`
- if present, read `docs/brand-guideline.json` — provides design tokens, Vuetify theme config (`vuetify_theme` section), component specs, and `agent_rules` that the planner and coder must follow for all frontend features
- read `agent/run-state.json` if present
- read recent `progress.jsonl` entries if recovery context is needed
- read `docs/project-lessons.md` if present

If the project is missing required setup, stop and report the blocker.

## Step 2 — Pick the Next Feature

Mirror `get_next_feature()`:
- skip features where `passes == true`
- choose the first feature whose `depends_on` entries all have `passes == true`
- if none are ready:
  - if all passed: go to project completion
  - otherwise report blocked or invalid features

Show:
- progress summary
- next feature id/name/phase
- mode: `coding` or `verification_fix`

## Step 3 — Plan If Needed

If `plan_required == true` and the feature plan / task board are missing or stale, run the planner.

If `plan_required` is missing, treat it as `true`.
If `plan_path` or `task_board_path` is missing/invalid, run planner first and repair the feature metadata before any coding.

Planner output must match the current emberforge tool contract:
- plan markdown at `feature.plan_path`
- task board JSON at `feature.task_board_path`
- if `integration_test_required == true`, integration test plan markdown at `feature.integration_test_plan_path`

The task board schema must match the real runtime:
- task entries use `files`, `verification`, `review_focus`, `status`, `notes`, `evidence`
- review checkpoints are separate records, not a fake review task

## Step 4 — Coding Session

Determine session mode:
- `verification_fix` when the latest progress snapshot says `recovery_mode == "verification_fix"` for this feature
- otherwise `coding`

**Before starting**: create a git checkpoint tag `pre-{feature_id}-{session_id_short}` so the session can be rolled back if needed.

Provide the coder with:
- the full feature object
- feature plan and task board
- relevant `docs_refs` and `user_story_refs`
- feature memory / recovery context when present
- relevant lessons from `docs/project-lessons.md`
- relevant gotchas from `references/gotcha-library.md`
- design images from `docs_refs` only when `DESIGN_ENABLED=true`
- browser tools only when `UI_DEBUG_ENABLED=true` and `UI_DEBUG_MODE=during`

**When `session_mode == "verification_fix"`**, additionally provide:
- the formatted verification failure context: for each failed check (up to 4), include: check name, command, failed test hints (parsed test names from output), and up to 1500 chars of stdout/stderr output
- read `agent/feature-memory/{feature_id}.json` for structured failure records: `name`, `command`, `output_excerpt`, `failed_tests`, `first_error`, `suspect_files`
- the coder must reproduce the failures FIRST, then make the smallest safe fix

Important:
- the coder works through the plan using the current emberforge contract
- tasks move via `update_feature_task`
- **after each task update, sync `agent/feature-memory/{feature_id}.json`** from the task board (update open_tasks, recent_completed_tasks, current_task_id)
- every completed task must carry evidence
- review notes are written via `record_review_checkpoint`
- non-obvious cross-feature lessons go to `docs/project-lessons.md` via `append_lesson`
- the coder may call `mark_feature_complete` only after all implementation gates are satisfied

## Step 5 — Completion Gates (checked AFTER coder claims complete, BEFORE verification)

After the coder calls `mark_feature_complete`, verify these gates before running verification:
- if `plan_required == true`, the plan exists
- if `integration_test_required == true`, the integration test plan exists
- the task board exists
- all planned tasks are `completed`
- if `review_required == true`, review checkpoints are present

Also remember the stricter `mark_feature_complete` gate:
- completed tasks must include evidence
- unresolved verification failures in feature memory block completion

## Step 6 — Verification

Run the real verification sequence described in `references/verification.md`:
1. feature `test_command`
2. repo-wide regression env commands:
   - `VERIFY_LINT_CMD`
   - `VERIFY_TYPE_CMD`
   - `VERIFY_BUILD_CMD`
   - `VERIFY_REGRESSION_CMD_*`
3. feature regression `verify_commands`
4. repo-wide capability env commands:
   - `VERIFY_CAPABILITY_CMD_*`
5. feature capability `verify_commands`
6. UI verification when `UI_DEBUG_ENABLED=true` and `feature.ui_verify` is `smoke` or `module`
7. design-reference availability note when `DESIGN_ENABLED=true` and image refs exist

Repo-wide frontend commands may be skipped until a frontend scaffold exists.

On failure:
- keep the feature unpassed
- **write structured failure records to `agent/feature-memory/{feature_id}.json`**: for each failed check, record `name`, `command`, `output_excerpt` (up to 1500 chars), `failed_tests` (parsed test names), `first_error`, `suspect_files` — this is what the next `verification_fix` session reads
- set `recovery_mode = "verification_fix"` and `verification_feature_id = feature.id` in progress state
- preserve the raw verification results in progress (for formatted injection into the next session)
- record a `verification_report` event to `progress.jsonl` with `passed: false`
- write a session handoff to `agent/session-handoffs/` with `result: "verification_failed"`
- regenerate `emberforge-report.html` (Step 8) to reflect the failure
- next run for the same feature must enter `verification_fix`

On success:
- mark `passes=true` for the feature in `agent/features.json`
- **clear verification failures** from `agent/feature-memory/{feature_id}.json`
- clear `recovery_mode` and `verification_feature_id` in progress state
- record a `verification_report` event with `passed: true` and a `session_finished` event
- write a session handoff to `agent/session-handoffs/` with `result: "completed"`
- regenerate `emberforge-report.html` (Step 8) to reflect progress
- present the updated report to the user in the main session
- continue to the next ready feature unless the user asked for one-feature mode

On incomplete (coder did not call `mark_feature_complete` — ran out of turns or stopped early):
- do NOT run verification
- if current `session_mode == "verification_fix"`: **preserve** `recovery_mode` and `verification_feature_id` — the next run must continue the fix attempt
- if current `session_mode == "coding"`: **clear** recovery fields — this was a fresh coding attempt, not a fix
- write a session handoff to `agent/session-handoffs/` with `result: "incomplete"`
- record a `session_finished` event with `completed: false`
- present the partial progress to the user and explain what was left unfinished

## Step 7 — Project Completion

When all features pass:
- generate `{PROJECT_DIR}/README.md`
- summarize completed phases/features
- if final review is requested, run the reviewer and write `docs/reviewer-report.md`
- generate the final `emberforge-report.html` (see Step 8)

## Step 8 — Report Generation

Generate `{PROJECT_DIR}/emberforge-report.html` using the template and aggregation logic in `references/reporting.md`.

### When to generate

1. **After each feature completes verification** (pass or fail) — so the user always has a current report.
2. **At project completion** — final report with all features done.
3. **On user request** — if the user asks for status or report mid-run.

### How to generate

1. Read `agent/features.json` — get all features and their `passes` status.
2. Compute schedule status for each feature (done / ready / blocked / invalid) by resolving the `depends_on` graph.
3. Parse `progress.jsonl` — aggregate `session_finished` / `planning_session_finished` events for global and per-feature token/cost totals.
4. Build the self-contained HTML string using the template in `references/reporting.md`.
5. Write the file to `{PROJECT_DIR}/emberforge-report.html` using the Write tool.

### Presenting to the user

After writing the report, always output to the main session:
- The report file path: `{PROJECT_DIR}/emberforge-report.html`
- A text summary: progress bar, e.g., `3/6 features done (F01 ✓, F02 ✓, F03 ✓, F04 ready, F05 blocked, F06 blocked)`
- Any blockers or failures that need attention

## Rules

1. Prefer the current emberforge tool contract over older skill wording.
2. Never invent or rely on `.skill-state.json` / `.skill-config` as the canonical runtime.
3. Never write to `skills/emberforge/**` during a project run.
4. Keep all project-relative paths scoped to `PROJECT_DIR`.
5. Design references and UI screenshots are project-scoped, not emberforge-root scoped.
6. Do not mark a feature passed without successful verification.
7. If you discover a mismatch between old skill docs and the current emberforge runtime, follow the runtime and note the mismatch.
8. **Never delegate work to external coding tools or agents** — do not invoke Codex, Claude Code, Aider, Cursor Agent, OpenAI API, or any other external code-generation service. All planning, coding, and verification must be performed directly by the skill within the current VS Code session using only the available workspace tools.
