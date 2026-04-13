# emberforge Workflow Reference

This is the current lifecycle the skill must mirror.

## Initialization (Two-Phase)

Before the core feature loop, the emberforge runs a two-phase initializer (once per project):

**Phase 1 — Environment Setup** (feature_id `INIT_PHASE1`):
- Verify Python, Node, git; init git if needed
- Quick-scan `agent/features.json` and `development-plan.md`
- Create `.venv` via `uv venv` (no package installs)
- Create initial git commit with project scaffold
- Mark `INIT_PHASE1` complete

**Phase 2 — System Configuration** (feature_id `INIT_PHASE2`):
- **Template auto-loading** runs before the Phase 2 agent session: `template_loader.auto_load_templates()` collects all `tech_tags` from features, matches them against `docs/templates/template-*.md` frontmatter, copies matches to project `docs/`, and appends paths to each feature's `docs_refs`
- Verify auto-loaded templates in `docs/`, fix `docs_refs` links
- Validate `features.json`: path rules + verify loop audit per feature type
- Populate `docs/architecture.md`, `docs/conventions.md`, `docs/README.md`
- Final git commit
- Mark `INIT` complete; set `git_initialized = true`

Both phases are strict about NOT installing packages or creating application code.
If either phase fails, the emberforge stops and requires re-running `--mode init`.

## Invocation Rules

Any user request that implies development work (including short prompts like `help me dev xxx`) must start from the standard lifecycle, not direct coding.

Required order:
1. Resolve target project and mode
2. Run two-phase initialization when required (`agent/run-state.json` missing or `git_initialized=false`)
3. Run preflight and read runtime artifacts (`agent/features.json`, `agent/run-state.json`, `progress.jsonl`, feature memory when present)
4. Select next dependency-ready feature
5. Run planning before coding when `plan_required=true`, or when planning fields/artifacts are missing

Planning fallback:
- missing `plan_required` is treated as `true`
- missing/invalid `plan_path` or `task_board_path` must be repaired by planning before coding starts

## Core Loop

For each dependency-ready feature:
1. plan if needed
2. implement task-by-task
3. satisfy completion gates
4. run verification
5. on success: mark `passes=true`
6. on failure: persist recovery context and re-enter `verification_fix`
7. **record events** to `progress.jsonl` throughout
8. **regenerate `emberforge-report.html`** after each verification (pass or fail)

## Feature Lifecycle

```text
pending -> planning -> coding -> verifying -> passed -> report
                              \
                               -> verification_fix -> verifying -> passed -> report
```

## Scheduling

Mirror `get_next_feature()`:
- walk features in file order
- skip passed features
- pick the first feature whose dependencies have all passed
- if none are ready, report blocked/invalid state

## State Model

The current emberforge does not rely on a skill-private `.skill-state.json`.

Use the standard artifacts instead:
- `agent/features.json`
- `agent/run-state.json`
- `agent/feature-memory/Fxx.json`
- `agent/session-handoffs/*.json`
- `progress.jsonl`

`features.json` remains the single source of truth for feature completion.

## Planning Contract

Planning must produce artifacts compatible with:
- `write_feature_plan`
- `mark_plan_complete`

That means:
- concrete feature plan markdown
- task board JSON using `files`, `verification`, `review_focus`, `status`, `notes`, `evidence`
- integration test plan when required

## Coding Contract

Coding must match the current runtime prompt:
- read before editing
- update tasks as work progresses
- record evidence on completed tasks
- record review checkpoints
- self-review diffs before commit
- call `mark_feature_complete` only after implementation-side gates are satisfied

## Recovery Contract

When verification fails:
- the feature stays unpassed
- progress keeps `recovery_mode=verification_fix` and `verification_feature_id`
- **structured failure records are written to `agent/feature-memory/{fid}.json`** with: name, command, output_excerpt, failed_tests, first_error, suspect_files
- raw verification results are stored in `progress.verification_artifacts`
- the next run for the same feature must prioritize reproducing and fixing those failures
- formatted failure context (up to 4 checks, 1500 chars each) is injected into the coder's prompt

When verification succeeds:
- verification failures are **cleared** from feature memory
- `recovery_mode`, `verification_feature_id`, `verification_artifacts` are all cleared

When a session is incomplete (coder didn't call `mark_feature_complete`):
- if currently in `verification_fix`: recovery mode **persists** — next run retries the fix
- if currently in `coding`: recovery fields are **cleared** — next run starts fresh

## Reviewer Contract

The standard final-review artifact is:
- `docs/reviewer-report.md`

Do not use `docs/review-report.md`.

## Recording Contract

Append JSON lines to `progress.jsonl` using this skill's event contract:

- `session_started` / `planning_session_started` before work begins
- `feature_task_updated` on each task status change
- `feature_plan_written` after planning artifacts are written
- `review_checkpoint` after recording a review
- `verification_report` after running verification
- `session_finished` / `planning_session_finished` when work on a feature ends

Session handoffs go to `agent/session-handoffs/{feature_id}_{timestamp}.json`.

See `references/reporting.md` for full event schema and payload shapes.

## Report Generation

Generate `emberforge-report.html` at these points:
1. After each feature verification (pass or fail)
2. At project completion
3. On user request

The report reads `features.json` for status and `progress.jsonl` for token/cost aggregation, then writes a self-contained HTML file.

See `references/reporting.md` for the HTML template and aggregation logic.
