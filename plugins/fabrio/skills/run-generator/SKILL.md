---
name: run-generator
description: "Runs a recurring generator job once — walks its saved step tree to read a source, dedup against open tickets, and file a task per new issue."
---

# Run Generator

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:ops-heartbeat`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

Execute one cycle of a **generator** job: walk its saved **step tree** to pull work from the job's source, drop anything that already has an open ticket, and file a task per remaining issue — recording what each step did as you go, so a failed run points at the step that stopped it. A generator never "completes"; it keeps producing tickets on its cadence.

**Invocation:** `$fabrio:run-generator <item_number>` — the job's human id (`#N`). Also dispatched automatically by `$fabrio:ops-heartbeat` for due generators.

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools). If the tools aren't available, stop and tell the user the server isn't connected — give them exactly this, and nothing else:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> export FABRIO_API_KEY="fab_live_YOUR_KEY"\ncodex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY
> ```
> Restart Codex, open a new task, and re-invoke.

---

## Step 1 — Fetch the job + its procedure

Call `get_plan_item { item_number }`. Capture:
- `id` — the job's UUID; **use it as `plan_item_id` for the write tools below**.
- `kind` — must be `generator`. If not, stop: this skill only runs generators.
- `steps` — the job's nested procedure. Each node has `id`, `path` (`"5.2"`), `title`, `instructions`, `step_type`, `output_key`, `foreach_source`, `max_iterations`, `condition`, `children`.
- `job_plan` — the compiled rendering of those steps, or hand-written prose for a job authored before steps existed.
- `open_tasks` — tickets already spawned from this job that are still open. **This is your dedup set.**
- `department`, site context — for shaping tickets.

**Pick a mode:**
- `steps` non-empty → **step mode** (Steps 1.6–2).
- `steps` empty but `job_plan` set → **legacy prose mode** (Step 2.9).
- Neither → stop, and tell the user to run `$fabrio:plan-job {item_number}` first.

---

## Step 1.4 — Open the run

`open_generator_run { plan_item_id: id, steps_total: <count of ALL nodes in the tree, children included — omit in legacy prose mode> }` → capture `run_id`. Do this **before** any work, including the preflight below, so every outcome has somewhere to attach and a run that dies mid-way still leaves a trace.

---

## Step 1.5 — Resolve and preflight the job's resource

If the **Fetch** step's `instructions` open with a resource line — `resource: <id> (<provider> · <access_method> · mcp: <server>)` — resolve it **before doing any work**. (In legacy prose mode, look for the same line in `job_plan`.)

1. Call `get_resource { resource_id }`. If the plan names only a category, call `list_site_resources { site_id: <the job's site>, resource_type: "<type>" }` and take the first match. A resource the plan names that no longer exists counts as unreachable.

2. **Preflight by `access_method` — check, never prompt:**
   - `remote_mcp` / `local_mcp` → are the named server's tools present in this session (`mcp__<mcp_server_name>__*`)? A server that's registered but not signed in errors on first call — treat that as `unauthenticated`. When `per_site_credentials` is true the server is registered **per site** and `mcp_server_name` already carries the site suffix — use it verbatim. If it still contains a literal `{{site}}`, you resolved the resource without a site: re-fetch with `list_site_resources { site_id }`.
   - `http_api` and `dashboard_link` → **reference-only.** Neither is agent-queryable: Fabrio records the endpoint and which credentials are needed, but not how to call it, so there is nothing to preflight and a passing credential check would be a false green. If a Fetch step depends on one, fail with `"Source unavailable: <name> is reference-only"` and say to re-run `$fabrio:plan-job {item_number}`. (Exception: if the resource's **notes** spell out the exact calls, follow them — but treat a missing name from `credential_keys` in `~/.fabrio/credentials.json` as `unauthenticated`. Read only the key **names**; never echo a value into output, a task, or a run summary.)

3. **On success**, call `record_resource_check { resource_id, status: "ok" }` and continue. This is the only way the Fabrio UI ever learns this machine can reach the resource — the browser can't see your local MCP servers or credentials file.

4. **On failure, stop cleanly. Never prompt** — `$fabrio:ops-heartbeat` dispatches this skill through Codex agent delegation and a prompt would hang the whole cycle. Make the calls below, then report and stop:
   - `record_resource_check { resource_id, status: "unreachable" | "unauthenticated", detail: "<one line + the exact fix>" }`
   - in step mode, `record_job_step { run_id, job_step_id: <the preflight step>, status: "failed", summary: "<provider> unreachable" }`
   - `record_generator_run { plan_item_id: id, run_id, items_found: 0, tasks_created: 0, status: "failed", failed_step_id: <the preflight step>, summary: "Source unavailable: <provider> — <detail>" }`

   Put copy-paste remediation in `detail` so the UI can show it verbatim, e.g. `"MCP server 'sentry' not connected on this machine — run: codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY` The `get_resource` result carries `setup_command` — use it rather than inventing one.

---

## Step 2 — Walk the step tree

Work the root steps in order. Keep one scratchpad in context:

```
outputs = { <output_key>: <what that step produced>, ... }
```

When a step declares an `output_key`, file its result there and **state the count out loud** ("`clusters` = 14 error clusters") so a long run can't lose it. Steps without an `output_key` produce transient results. None of this goes to the database — only one-line receipts do.

**`action`** — do what `instructions` says. Then `record_job_step { run_id, job_step_id: <node id>, status: "completed", summary: "<one line>" }`.

**`foreach`** — resolve the collection:
1. `foreach_source: "step:<key>"` → `outputs[key]`. If that key was never produced, the step fails.
2. `foreach_source` absent → the output of the **immediately preceding sibling step**. Restate that list as short labels *before* entering the loop.

Cap iterations at `max_iterations`, else **10**. If the collection is over the cap, drop the excess and name what you dropped in the summary — a truncated run must never read as "nothing else found". If it's empty, record the node `completed` with `summary: "0 items"` and skip its children.

Run the node's `children` in order for each element. Then record **one** receipt for the foreach node itself: `summary: "12 items · 10 ok · 2 failed"`.

**`branch`** — evaluate `condition`. True → run its `children`. False → `record_job_step { …, status: "skipped" }` and do **not** descend or record its children.

### Recording discipline

These bounds are the difference between useful run history and a write storm:
1. One receipt per **root** step, always — this is what makes "stopped at step 4" work.
2. One receipt per **foreach node**, summarising the whole loop.
3. Inside a loop, a receipt per **iteration only when that iteration FAILED** (`iteration`, `iteration_label`). Never one per child per iteration.
4. `summary` is one short line, ~200 chars. It is a receipt, not a transcript — no data dumps, no stack traces, and never a credential value or PII lifted from a log.

### Filing tickets

Ticket-filing normally lives in a step inside the `foreach`. For each new issue call `create_generator_task { plan_item_id: id, title, description, ... }`:
- `title` — concise, per the step's title format.
- `description` — carry the evidence the step specified (IDs, counts, affected areas, links, repro steps) so the downstream implementer has everything.
- Optionally `acceptance_criteria` and `difficulty`.

Each call creates one `ready` task linked back to this job. Dedup against `open_tasks` first — matching on title + gist — and skip anything already covered. That is what makes repeated runs safe.

### When something fails

| what failed | what to do |
|---|---|
| a **root** step | record it `failed`, **abort the remaining root steps**, and close the run with `failed` + `failed_step_id`. Steps never reached get no receipt — absence *is* "not reached". |
| one **iteration** of a foreach | record that iteration `failed` and **continue the loop**. One bad item must not cost you the other nine tickets. |
| the **source**, mid-run (expired OAuth, rate limit, revoked key) | treat as a root-step failure, and also `record_resource_check` with the real status, as in Step 1.5 §4. |

Never prompt — `$fabrio:ops-heartbeat` dispatches this skill through Codex agent delegation. Every exit is a durable artifact.

---

## Step 2.9 — Legacy prose mode

A job with no `steps` still has hand-written `job_plan` prose. Follow it as written, end to end, in one pass. Still call `open_generator_run` (with no `steps_total`) and still close with `record_generator_run { run_id, … }` — you simply record no per-step receipts. Mention in your output that `$fabrio:plan-job {item_number}` would convert it to steps.

---

## Step 5 — Close the run (advances the cadence)

Call **exactly once** at the end, even if zero tasks were filed and even if the run failed:

`record_generator_run { plan_item_id: id, run_id, items_found: <candidates considered>, tasks_created: <new tickets filed>, status: "completed" | "failed", failed_step_id: <the step that stopped it, if any>, summary: "<one line: found X, filed Y, skipped Z dupes>" }`

This closes the run row the UI shows **and** advances `next_run_at` by the job's cadence — on failure too, so the job retries next period instead of wedging. Do not call `queue_plan_item_task` for a generator — that path is for execution items only.

---

## Step 6 — Output summary

```
🔁 Generator run: "{item.title}"
  Steps:             {completed}/{total}   {✓ | ✗ stopped at {path} — {title}}
  Candidates found:  {items_found}
  New tickets filed: {tasks_created}   (skipped {dupes} already-open)
  Next run:          {next_run_at}
```
