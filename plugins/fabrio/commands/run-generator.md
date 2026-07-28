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

## Step 1.5 — Resolve and preflight the job's resource

If the `job_plan`'s **Fetch** step opens with a resource line — `resource: <id> (<provider> · <access_method> · mcp: <server>)` — resolve it **before doing any work**:

1. Call `get_resource { resource_id }`. If the plan names only a category, call `list_site_resources { site_id: <the job's site>, resource_type: "<type>" }` and take the first match. A resource the plan names that no longer exists counts as unreachable.

2. **Preflight by `access_method` — check, never prompt:**
   - `remote_mcp` / `local_mcp` → are the named server's tools present in this session (`mcp__<mcp_server_name>__*`)? A server that's registered but not signed in errors on first call — treat that as `unauthenticated`. When `per_site_credentials` is true the server is registered **per site** and `mcp_server_name` already carries the site suffix — use it verbatim. If it still contains a literal `{{site}}`, you resolved the resource without a site: re-fetch with `list_site_resources { site_id }`.
   - `http_api` and `dashboard_link` → **reference-only.** Neither is agent-queryable: Fabrio records the endpoint and which credentials are needed, but not how to call it, so there is nothing to preflight and a passing credential check would be a false green. If a Fetch step depends on one, fail with `"Source unavailable: <name> is reference-only"` and say to re-run `/fabrio:plan-job {item_number}`. (Exception: if the resource's **notes** spell out the exact calls, follow them — but treat a missing name from `credential_keys` in `~/.fabrio/credentials.json` as `unauthenticated`. Read only the key **names**; never echo a value into output, a task, or a run summary.)

3. **On success**, call `record_resource_check { resource_id, status: "ok" }` and continue. This is the only way the Fabrio UI ever learns this machine can reach the resource — the browser can't see your local MCP servers or credentials file.

4. **On failure, stop cleanly. Never prompt** — `/fabrio:ops-heartbeat` dispatches this skill headlessly and a prompt would hang the whole cycle. Make **both** calls, then report and stop:
   - `record_resource_check { resource_id, status: "unreachable" | "unauthenticated", detail: "<one line + the exact fix>" }`
   - `record_generator_run { plan_item_id: id, items_found: 0, tasks_created: 0, status: "failed", summary: "Source unavailable: <provider> — <detail>" }`

   Put copy-paste remediation in `detail` so the UI can show it verbatim, e.g. `"MCP server 'sentry' not connected on this machine — run: claude mcp add --transport http sentry https://mcp.sentry.dev/mcp, restart Claude Code, approve the OAuth prompt."` The `get_resource` result carries `setup_command` — use it rather than inventing one.

---

## Step 2 — Follow the plan to gather candidates

Execute the **Fetch** and **Select & rank** steps exactly as written in `job_plan`, using whatever source it names (MCP tools, HTTP API, etc.) — for a resource-backed job that's the one resolved in Step 1.5, scoped by its per-site config (`service`, `env`, `project_slug`, `app_id`, …). Produce a ranked, top-N list of candidate issues.

If the source fails **mid-fetch** — it preflighted fine but then errored (expired OAuth, rate limit, revoked key) — take the same clean exit as Step 1.5 §4: `record_resource_check` with the real status, then `record_generator_run { plan_item_id: id, items_found: 0, tasks_created: 0, status: "failed", summary: "Source unavailable: <detail>" }`, and report it. This still advances the cadence so the job retries next period — do not leave it wedged, and do not prompt.

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
