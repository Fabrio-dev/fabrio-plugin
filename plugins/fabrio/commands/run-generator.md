---
description: "Runs a recurring generator job once — follows its saved plan to read a source, dedup against open tickets, and file a task per new issue."
---

# Run Generator

Execute one cycle of a **generator** job: follow its saved `job_plan` to pull work from the job's source, drop anything that already has an open ticket, and file a task per remaining issue. A generator never "completes" — it keeps producing tickets on its cadence.

**Invocation:** `/fabrio:run-generator <item_number>` — the job's human id (`#N`). Also dispatched automatically by `/fabrio:ops-heartbeat` for due generators.

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools). If unavailable, stop and tell the user to connect it (**Settings → API keys** → Connect command), then re-invoke.

---

## Step 1 — Fetch the job + its plan

Call `get_plan_item { item_number }`. Capture:
- `id` — the job's UUID; **use it as `plan_item_id` for the write tools below**.
- `kind` — must be `generator`. If not, stop: this skill only runs generators.
- `job_plan` — the procedure to follow. If empty, stop and tell the user to run `/fabrio:plan-job {item_number}` first.
- `open_tasks` — tickets already spawned from this job that are still open. **This is your dedup set.**
- `department`, site context — for shaping tickets.

---

## Step 2 — Follow the plan to gather candidates

Execute the **Fetch** and **Select & rank** steps exactly as written in `job_plan`, using whatever source it names (MCP tools, HTTP API, etc.). Produce a ranked, top-N list of candidate issues.

If the source can't be reached (missing MCP / credentials the plan flagged as required), stop gracefully: call `record_generator_run { plan_item_id: id, items_found: 0, tasks_created: 0, status: "failed", summary: "Source unavailable: <detail>" }` and report it. This still advances the cadence so the job retries next period — do not leave it wedged.

---

## Step 3 — Dedup

For each candidate, check `open_tasks` (title + gist) for an existing open ticket covering the same issue. **Skip matches** — don't re-file. Only genuinely new issues proceed. This is what makes repeated runs safe.

---

## Step 4 — File a task per new issue

For each new issue, call `create_generator_task { plan_item_id: id, title, description, ... }`:
- `title` — concise, per the plan's title format.
- `description` — carry the evidence the plan specified (IDs, counts, affected areas, links, repro steps) so the downstream implementer has everything.
- Optionally `acceptance_criteria` and `difficulty`.

Each call creates one `ready` task linked back to this job (`plan_item_id`). These flow through the normal loop — a later heartbeat implements them, or a human triages.

---

## Step 5 — Record the run (advances the cadence)

Call **exactly once** at the end, even if zero tasks were filed:
`record_generator_run { plan_item_id: id, items_found: <candidates considered>, tasks_created: <new tickets filed>, summary: "<one line: found X, filed Y, skipped Z dupes>" }`.

This writes the run history the UI shows **and** advances `next_run_at` by the job's cadence. Do not call `queue_plan_item_task` for a generator — that path is for execution items only.

---

## Step 6 — Output summary

```
🔁 Generator run: "{item.title}"
  Candidates found:  {items_found}
  New tickets filed: {tasks_created}   (skipped {dupes} already-open)
  Next run:          {next_run_at}
```
