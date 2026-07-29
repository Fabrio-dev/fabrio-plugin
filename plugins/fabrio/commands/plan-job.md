---
description: "Turns a recurring job's description into a saved, reusable tree of steps it re-runs each cadence — asking blocking questions when it needs human input."
---

# Plan Job

Every recurring **job** has a human `description` (what it does / how it gathers work) and a required, AI-authored **procedure** — an ordered, nested tree of **steps** it re-runs each cadence. This skill compiles the description into those steps. It mirrors how a task has a human `description` + an AI `task_plan`, and — like `/fabrio:feature-request` — it can **ask a blocking question** and wait for a human when a real choice is needed.

Steps replaced the old single `job_plan` text blob so that a job's stages have somewhere to live. Before, they had nowhere to go — so plans emitted them as sibling initiatives instead, and a job could end up "blocked by" a stage of its own procedure. `job_plan` still exists, but it is now *generated* from the steps.

**Invocation:** `/fabrio:plan-job <item_number>` — the job's human id, shown as `#N` on the job in the plan UI.

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools). If the tools aren't available, stop and tell the user the server isn't connected — give them exactly this, and nothing else:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
> ```
> Restart Claude Code and re-invoke.

---

## Step 1 — Fetch the job

Call `get_plan_item { item_number }`. Capture:
- `id` — the job's UUID; **use it as `plan_item_id` for every write tool below** (update_plan_item, create_task_question, create_decision links).
- `description` — the human intent. **If empty, stop** and tell the user to add a description to the job first (e.g. "Each week, pull open bugs from our Jira project ABC, take the top 5 by severity, and file a ticket for each").
- `is_blocked` / existing questions — see Step 2.
- `kind` (`execution` | `generator`), `frequency`, `department`, `execution_mode`, and site context (`plan.site.name`, `live_url`, `ai_context`).

**`execution_mode` shapes the whole procedure**, so settle it before writing steps (Step 3.5). A `repo` job ends in a branch + PR; an `artifact` job ends in a saved markdown deliverable; an `external` job ends in a ready-to-execute package a human performs. Writing repo/PR steps for a job that publishes to a social channel produces a plan that can never run.

Then call `list_site_resources { site_id: plan.site_id }` — the third-party tools this site is wired to (monitoring today; analytics/hosting later). Each entry gives `provider`, `access_method`, the non-secret connection recipe, the per-site config (`service`, `env`, `project_slug`, `app_id`, …), `mcp_server_name`, and `credential_keys` — the **names** of keys a machine must have in `~/.fabrio/credentials.json`. Fabrio never stores the values. **This is your source list: prefer a real attached resource over asking a human.**

---

## Step 2 — Check for open questions (don't plan while blocked)

If the job already has an **open question** (`is_blocked` true), stop:
> ⏸ This job is waiting on an answer. Answer it in the job's Questions tab, then re-run `/fabrio:plan-job {item_number}`.

Then load prior **decisions** so you don't re-ask what's settled: `list_decisions { site_id: plan.site_id, status: "decided" }`. A `decided` decision is binding — apply its `chosen_option_key`/`chosen_rationale`.

---

## Step 3 — Decide whether you need human input

Read the `description` and decide: **can you write a concrete, runnable plan without guessing at something a human should decide?** Real decisions worth asking about: *which* tracker/tool when the description is vague, *which* project/board/label, the top-N cap, or which department the filed tickets belong to. Do **not** ask about things you can reasonably choose or that a `decided` decision already covers.

**Resources answer the old "where do credentials come from?" question — stop asking it.** If Step 1 returned a resource whose type/provider matches what the description names, use it and record its id in the plan. Only ask when:
- the site has **no** attached resource for the category the description needs (e.g. "pull error rates" with nothing of type `monitoring` attached), or
- **two or more** attached resources of that type could plausibly serve it.

In the first case the fix is a human action in the web UI, not an answer — say so verbatim so it's actionable, and stop:
> This job needs a **monitoring** resource for {site.name}, but none is attached. Add one in **Fabrio → Resources**, attach it to this site, then re-run `/fabrio:plan-job {item_number}`.

In the second case, prefer a structured `create_decision` whose `options` are the candidate resources (`key` = resource id, `label` = "{provider} — {name}").

If you need input, open a blocking question **on the job** (use the `id` from Step 1 as `plan_item_id`) and stop:
- **Structured decision** (preferred when there are clear options): `create_decision { site_id, key, title, description, options:[…] }` (idempotent), then `create_task_question { plan_item_id: id, content, decision_id }`.
- **Freeform**: `create_task_question { plan_item_id: id, content }`.

Then stop:
> ⏸ Clarification needed to plan this job — answer in the Questions tab, then re-run `/fabrio:plan-job {item_number}`.

(Opening a question auto-blocks the job; `list_due_plan_items` skips it until answered.)

---

## Step 3.5 — Settle the execution mode

If `item.execution_mode` from Step 1 is already set, honour it. If it's null, decide from the `description` — **where does each run's output end up?**

- **`repo`** — files in a site repo. A weekly blog post committed to `content/posts/`, a monthly dependency bump, generated sitemap entries. Steps end in a branch + PR.
- **`artifact`** — a document with no repo home. A monthly performance report, a refreshed keyword backlog, a competitor scan. Steps end in a saved markdown deliverable.
- **`external`** — an action on a third-party system. Posting to social, sending a campaign, adjusting ad spend, outreach. Steps end in a ready-to-execute package; **a human performs the action, always.**
- A **generator** job (`kind: "generator"`) files tickets rather than producing the work itself, so its own mode describes *the tickets it files* — usually `repo` for a bug-triage job. The filed tickets inherit it.

Persist it so queued tasks route correctly: `update_plan_item { plan_item_id: id, fields: { execution_mode: "{mode}" } }`. Say which mode you chose and why in the Step 6 summary.

A job whose description mixes modes ("write the post **and** promote it on social") is two initiatives, not one — file a question rather than authoring a plan that ends in an action Fabrio may not take:
`create_task_question { plan_item_id: id, content: "This job mixes producing content (a PR) with publishing it externally. Should I split it into two plan items?" }` and stop.

---

## Step 4 — Design the step tree

A job's procedure is an ordered, **nested** tree of steps, not one blob of prose. Draft that tree. **Let the mode from Step 3.5 drive the closing steps** — a `repo` job ends in a PR, an `artifact` job in a saved deliverable, an `external` job in a prepared package. Never author a step that publishes, sends, or spends: those belong to the human.

**What makes a step.** One step = one verb over one input. Nest a step **under** another when it must run *once per item* of that step's output — that is a `foreach`. Use a `branch` when a step only sometimes applies. Every leaf must be executable without re-reading the original description.

**Step types**
- `action` (default) — do one thing.
- `foreach` — repeat its `children` once per item of `foreach_source` (`"step:<output_key>"`, or omit to iterate the immediately preceding step's output). Set `max_iterations` to cap a run; the runner defaults to 10.
- `branch` — run its `children` only when `condition` holds.

Give a step an `output_key` (snake_case) when a later step needs to name its result.

**Put the detail in `instructions`, not in the titles.** Concrete queries, field names, path mappings, title formats, redaction rules, noise filters — all of it belongs in the owning step's `instructions`, verbatim. A tree of thin titles is a regression against a good prose plan.

**Limits:** depth 3, 12 top-level steps, 60 total. If you need more, fold detail into `instructions` rather than adding levels.

**Cover these concerns across the tree:**
1. **Fetch** — how to pull candidate items from the source. When the source is an attached resource, **open that step's `instructions` with a machine-readable line** so `/fabrio:run-generator` can preflight it before doing any work:

   `resource: <resource_id> (<provider> · <access_method> · mcp: <mcp_server_name>)`

   Then spell out the concrete call — which MCP tool or HTTP endpoint, and the query/filter, scoped by the resource's per-site config (e.g. `service:{config.service} env:{config.env}` for Datadog, `project:{config.project_slug}` for Sentry). **Never inline a credential value** — name the `credential_keys` and let the runner read them from `~/.fabrio/credentials.json`. If the resource exists in Fabrio but nobody has connected it on a machine yet, still author the steps: `/fabrio:run-generator` preflights and fails cleanly with the setup command rather than half-running.
2. **Select & rank** — how to cluster/rank and how many to file per run (a top-N cap).
3. **Dedup** — skip issues that already have an open ticket from this job (`/fabrio:run-generator` gets `open_tasks` from `get_plan_item`). Make this its own step *before* the loop that files tickets.
4. **Shape each ticket** — title format + what evidence goes in the description (IDs, counts, links, repro). This is normally a step inside the `foreach`.

Do **not** add a "record the run" step — the runner records every run itself.

**Shape to aim for:**
```
1. Preflight the monitoring resource                        action   → resource
2. Fetch and cluster errors for the window                  action   → clusters
3. Filter known noise                                       action   → candidates
4. Rank and cap to the top 3                                action   → selected
5. For each selected issue                foreach (step:selected, max 3)
   5.1 Investigate the codebase for the fault               action
   5.2 File a ticket with the remediation plan              action
```

---

## Step 5 — Save it

Call `replace_job_steps { plan_item_id: id, steps: [ … ] }` (use the `id` from Step 1). It replaces the job's whole tree, so send every step each time.

For a ticket-filing job also call `update_plan_item { plan_item_id: id, fields: { kind: "generator" } }` if it isn't one already. A simple recurring task that just needs a documented procedure keeps `kind: "execution"` — it still benefits from steps.

**Do not write `job_plan`.** It is generated from the steps server-side, and `update_plan_item` rejects the field once a job has any.

If `replace_job_steps` isn't available in this session, the connected Fabrio predates nested steps: fall back to `update_plan_item { plan_item_id: id, fields: { job_plan: "<the procedure as numbered prose>" } }` and tell the user to update Fabrio.

---

## Step 6 — Output summary

```
✅ Job procedure saved for "{item.title}" ({frequency} · {department} · {execution_mode})
{N} steps{, including a foreach over {what}}
Source: {how it fetches work — e.g. "Jira project ABC via MCP"}
Output: {repo: "a PR per run" | artifact: "a markdown deliverable per run" | external: "a ready-to-execute package per run — you perform the action"}
{Files up to {N} tickets/run, deduped against open ones. | Queues one task/run.}

Run it now with /fabrio:run-generator {item_number} (generator) — or let the ops heartbeat run it on schedule.
```
