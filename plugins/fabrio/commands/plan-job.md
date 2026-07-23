---
description: "Turns a recurring job's description into a saved, reusable plan (job_plan) it re-runs each cadence — asking blocking questions when it needs human input."
---

# Plan Job

Every recurring **job** has a human `description` (what it does / how it gathers work) and a required, AI-authored **`job_plan`** — the exact procedure it re-runs each cadence. This skill compiles the description into that plan. It mirrors how a task has a human `description` + an AI `task_plan`, and — like `/fabrio:feature-request` — it can **ask a blocking question** and wait for a human when a real choice is needed.

**Invocation:** `/fabrio:plan-job <item_number>` — the job's human id, shown as `#N` on the job in the plan UI.

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools). If unavailable, stop and tell the user to connect it (**Fabrio → Settings → API keys** → run the Connect command), then restart Claude Code and re-invoke.

---

## Step 1 — Fetch the job

Call `get_plan_item { item_number }`. Capture:
- `id` — the job's UUID; **use it as `plan_item_id` for every write tool below** (update_plan_item, create_task_question, create_decision links).
- `description` — the human intent. **If empty, stop** and tell the user to add a description to the job first (e.g. "Each week, pull open bugs from our Jira project ABC, take the top 5 by severity, and file a ticket for each").
- `is_blocked` / existing questions — see Step 2.
- `kind` (`execution` | `generator`), `frequency`, `department`, and site context (`plan.site.name`, `live_url`, `ai_context`).

---

## Step 2 — Check for open questions (don't plan while blocked)

If the job already has an **open question** (`is_blocked` true), stop:
> ⏸ This job is waiting on an answer. Answer it in the job's Questions tab, then re-run `/fabrio:plan-job {item_number}`.

Then load prior **decisions** so you don't re-ask what's settled: `list_decisions { site_id: plan.site_id, status: "decided" }`. A `decided` decision is binding — apply its `chosen_option_key`/`chosen_rationale`.

---

## Step 3 — Decide whether you need human input

Read the `description` and decide: **can you write a concrete, runnable plan without guessing at something a human should decide?** Real decisions worth asking about: *which* tracker/tool when the description is vague, *which* project/board/label, where credentials come from, the top-N cap, or which department the filed tickets belong to. Do **not** ask about things you can reasonably choose or that a `decided` decision already covers.

If you need input, open a blocking question **on the job** (use the `id` from Step 1 as `plan_item_id`) and stop:
- **Structured decision** (preferred when there are clear options): `create_decision { site_id, key, title, description, options:[…] }` (idempotent), then `create_task_question { plan_item_id: id, content, decision_id }`.
- **Freeform**: `create_task_question { plan_item_id: id, content }`.

Then stop:
> ⏸ Clarification needed to plan this job — answer in the Questions tab, then re-run `/fabrio:plan-job {item_number}`.

(Opening a question auto-blocks the job; `list_due_plan_items` skips it until answered.)

---

## Step 4 — Write the job_plan

When you have enough, draft a numbered, reusable procedure the job follows **every** run. Cover:
1. **Fetch** — exactly how to pull candidate items from the source named in the description (which MCP tool / HTTP endpoint + query/filter). If the source isn't reachable yet, still write the plan but mark the fetch step **requires <X> to be connected** so `/fabrio:run-generator` fails cleanly.
2. **Select & rank** — how to cluster/rank and how many to file per run (a top-N cap).
3. **Shape each ticket** — title format + what evidence to put in the description (IDs, counts, links, repro).
4. **Dedup** — skip issues that already have an open ticket from this job (`/fabrio:run-generator` gets `open_tasks` from `get_plan_item`).
5. **Record** — end each run with `record_generator_run`.

Keep it self-contained: executable without re-reading the original description.

---

## Step 5 — Save it

Call `update_plan_item { plan_item_id: id, fields: { job_plan: "<the procedure>", kind: "generator" } }` (use the `id` from Step 1) for a ticket-filing job (set `kind: "generator"` if not already). For a simple recurring task that just needs a documented plan, save `job_plan` and leave `kind: "execution"`.

---

## Step 6 — Output summary

```
✅ Job plan saved for "{item.title}" ({frequency} · {department})
Source: {how it fetches work — e.g. "Jira project ABC via MCP"}
{Files up to {N} tickets/run, deduped against open ones. | Queues one task/run.}

Run it now with /fabrio:run-generator {item_number} (generator) — or let the ops heartbeat run it on schedule.
```
