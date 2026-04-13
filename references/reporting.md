# Reporting & Recording Reference

This file describes the progress recording and HTML report generation contract for this skill.

## progress.jsonl — Event Log

The skill must append events to `{PROJECT_DIR}/progress.jsonl` at key lifecycle points. Every record is a single JSON object on one line.

### Common Fields (every record)

```json
{
  "timestamp": "2026-04-12T10:30:00.000000",
  "event": "event_type_string",
  "session_id": "skill-<random-8-chars>",
  "feature_id": "F03"
}
```

Generate `session_id` once per skill invocation (e.g., `skill-a1b2c3d4`).

### Event Types the Skill Must Write

| Event | When | Extra Fields |
|---|---|---|
| `session_started` | Before starting work on a feature | `session_kind` ("planning" / "coding" / "verification_fix") |
| `planning_session_started` | Before planning a feature | — |
| `planning_session_finished` | After planning completes | `completed: true/false` |
| `feature_plan_written` | After writing plan + task board | `plan_path`, `task_board_path` |
| `feature_task_updated` | After each task status change | `task_id`, `old_status`, `new_status`, `evidence` (when completing) |
| `review_checkpoint` | After recording a review checkpoint | `task_id`, `stage`, `outcome`, `notes` |
| `verification_report` | After running verification | `passed: bool`, `results: [{name, command, passed, output_preview}]` |
| `session_finished` | After finishing work on a feature | See payload below |

### session_finished Payload

```json
{
  "event": "session_finished",
  "completed": true,
  "session_kind": "coding",
  "summary": "Implemented auth routes, all tests pass",
  "next_steps": "Proceed to F04",
  "plan_path": "docs/feature-plans/F03.md",
  "task_board_path": "agent/feature-tasks/F03.json",
  "total_input_tokens": 0,
  "total_output_tokens": 0,
  "estimated_cost_usd": 0.0
}
```

Note: The skill runs inside Claude Code so real token counts are not available. Write `0` for token fields. The report will still be valuable for feature status tracking.

### Writing Events

Use a Bash command or Write tool to append a single JSON line:

```bash
echo '{"timestamp":"...","event":"session_started",...}' >> progress.jsonl
```

Or build the JSON in-line and append. Always use append mode — never overwrite the file.

## Feature Schedule Analysis

The report computes each feature's status from `features.json`:

```
for each feature in features:
  if feature.passes == true:
    status = "done"
  else if all dependencies have passes == true:
    status = "ready"
  else if any dependency has passes == false AND is itself not blocked:
    status = "blocked"
    blockers = [dep_id for dep in depends_on if not dep.passes]
  else:
    status = "invalid"  (circular or missing dependency)
```

## HTML Report Template

Generate `{PROJECT_DIR}/emberforge-report.html` with this structure. The skill must build this as a single self-contained HTML string and write it via the Write tool.

```html
<!DOCTYPE html>
<html><head><title>Agent Progress Report</title>
<style>
  body { font-family: system-ui; max-width: 1100px; margin: 40px auto; padding: 0 20px; }
  .done { color: #22c55e; } .ready { color: #3b82f6; }
  .blocked { color: #f59e0b; } .invalid { color: #ef4444; }
  table { border-collapse: collapse; width: 100%; }
  td, th { border: 1px solid #ddd; padding: 8px; text-align: left; }
  th { background: #f9fafb; }
  td.num { text-align: right; font-variant-numeric: tabular-nums; }
  .summary { display: flex; gap: 24px; margin: 16px 0; flex-wrap: wrap; }
  .stat { padding: 12px 20px; background: #f0f9ff; border-radius: 8px; }
  .na { color: #aaa; }
</style></head><body>
<h1>Agent Progress Report</h1>
<p>Generated: {ISO_TIMESTAMP}</p>
<div class="summary">
  <div class="stat"><strong>{DONE_COUNT}/{TOTAL_COUNT}</strong> features done</div>
  <div class="stat"><strong>{TOTAL_TOKENS}</strong> total tokens</div>
  <div class="stat"><strong>${TOTAL_COST}</strong> estimated cost</div>
</div>
<h2>Feature Status</h2>
<table>
  <tr>
    <th>ID</th><th>Name</th><th>Phase</th><th>Status</th><th>Blockers</th>
    <th>Tokens (in)</th><th>Tokens (out)</th><th>Total Tokens</th><th>Cost (USD)</th>
  </tr>
  <!-- For each feature in schedule order: -->
  <tr>
    <td>{ID}</td><td>{NAME}</td><td>{PHASE}</td>
    <td class="{STATUS}">{STATUS_UPPER}</td><td>{BLOCKERS_OR_DASH}</td>
    <td class="num">{IN_TOKENS_OR_DASH}</td>
    <td class="num">{OUT_TOKENS_OR_DASH}</td>
    <td class="num">{TOTAL_TOKENS_OR_DASH}</td>
    <td class="num">{COST_OR_DASH}</td>
  </tr>
  <!-- After all features, if any token data exists: -->
  <tr style="font-weight:bold;background:#f9fafb">
    <td colspan="5">Total</td>
    <td class="num">{SUM_IN}</td>
    <td class="num">{SUM_OUT}</td>
    <td class="num">{SUM_TOTAL}</td>
    <td class="num">${SUM_COST}</td>
  </tr>
</table>
</body></html>
```

### Token/Cost Aggregation

Parse `progress.jsonl` for `session_finished` and `planning_session_finished` events:

- **Global totals**: Sum `total_input_tokens`, `total_output_tokens`, `estimated_cost_usd` across all such events.
- **Per-feature**: Group by `feature_id` and sum the same fields.
- Features with no session events show `<span class="na">—</span>` in the table.

### Report Rendering Rules

1. Use the feature order from `features.json` (not alphabetical).
2. Format token numbers with comma separators (e.g., `12,345`).
3. Format cost with 4 decimal places (e.g., `$0.1234`).
4. The status cell must use the CSS class matching the status: `done`, `ready`, `blocked`, `invalid`.
5. Blockers column: comma-separated dependency IDs, or `-` if none.
6. Only include the totals row if any token data exists.

## When to Generate the Report

The skill must generate the report at these points:

1. **After each feature completes verification** (pass or fail) — so the user always has current status.
2. **At project completion** (all features passed) — final report.
3. **On user request** — if the user asks for status/report mid-run.

After writing the file, always tell the user:
- The report path (`{PROJECT_DIR}/emberforge-report.html`)
- A text summary: `X/Y features done`, current feature status, any blockers

## Verification Result Recording

After running verification (Step 6 in SKILL.md), write a `verification_report` event:

```json
{
  "timestamp": "...",
  "event": "verification_report",
  "session_id": "skill-...",
  "feature_id": "F03",
  "passed": false,
  "results": [
    {
      "name": "Feature test",
      "command": "pytest backend/tests/test_auth.py -v",
      "passed": true,
      "output_preview": "6 passed in 0.8s"
    },
    {
      "name": "Frontend build",
      "command": "npm run build --prefix frontend",
      "passed": false,
      "output_preview": "Error: Cannot resolve module..."
    }
  ]
}
```

## Session Handoff Recording

At the end of each feature session (whether passed or failed), write a session handoff to `agent/session-handoffs/`:

```json
{
  "session_id": "skill-a1b2c3d4",
  "feature_id": "F03",
  "mode": "coding",
  "result": "completed",
  "current_task_id": "T05",
  "what_changed": ["backend/app/routers/auth.py", "backend/tests/test_auth.py"],
  "what_was_verified": ["pytest backend/tests/test_auth.py -v"],
  "open_risks": [],
  "resume_from": "",
  "must_read_files": [],
  "timestamp": "2026-04-12T10:45:00.000000"
}
```

File name pattern: `{feature_id}_{timestamp}.json` (e.g., `F03_20260412T104500.json`).
