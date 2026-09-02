---
name: plan-job
description: "Turns a recurring job's description into a saved, reusable tree of steps it re-runs each cadence — steps that plan the work and file the task that does it, asking blocking questions when it needs human input."
---

# Plan Job

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:run-due-jobs`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

Every recurring **job** has a human `description` (what it does / how it gathers work) and a required, AI-authored **procedure** — an ordered, nested tree of **steps** it re-runs each cadence. This skill compiles the description into those steps. It mirrors how a task has a human `description` + an AI `task_plan`, and — like `$fabrio:feature-request` — it can **ask a blocking question** and wait for a human when a real choice is needed.

**A job plans; a task does.** The steps you author read sources and decide what needs doing, then file a task — they never produce the work themselves. Step 3.4 states that contract in full, and it is the single most important thing on this page.

Steps replaced the old single `job_plan` text blob so that a job's stages have somewhere to live. Before, they had nowhere to go — so plans emitted them as sibling initiatives instead, and a job could end up "blocked by" a stage of its own procedure. `job_plan` still exists, but it is now *generated* from the steps.

**Invocation:** `$fabrio:plan-job <item_number>` — the job's human id, shown as `#N` on the job in the plan UI.

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools). If the tools aren't available, stop and tell the user the server isn't connected — give them exactly this, and nothing else:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> export FABRIO_API_KEY="fab_live_YOUR_KEY"\ncodex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY
> ```
> Restart Codex, open a new task, and re-invoke.

---

## Step 1 — Fetch the job

Call `get_plan_item { item_number }`. Capture:
- `id` — the job's UUID; **use it as `plan_item_id` for every write tool below** (update_plan_item, create_task_question, create_decision links).
- `description` — the human intent. **If empty, stop** and tell the user to add a description to the job first (e.g. "Each week, pull open bugs from our Jira project ABC, take the top 5 by severity, and file a ticket for each").
- `is_blocked` / existing questions — see Step 2.
- `frequency` — **if `one_time`, stop**: that's a one-off initiative, not a recurring job. It queues its task straight from the plan UI and needs no procedure.
- `department`, `execution_mode`, and site context (`plan.site.name`, `live_url`, `ai_context`). Then call `get_account_context` for the workspace's own `ai_context` — a job's steps hardcode provider and tool choices, so the portfolio-wide rules (which git host, which CMS, what may never be automated) are binding on the tree you write. It is the widest context layer; narrower layers (department playbook, site ai_context) win a direct conflict.
- The plan's **site set** (`plan.all_sites`, `plan.plan_sites`) and the job's own `site_id` override. A multi-site plan fans each filed task out to every targeted site — don't hardcode one site into the steps.

**`execution_mode` describes the TASKS this job files, not the job's own steps** — settle it before writing them (Step 3.5). It never changes how the tree ends: every tree ends in a `create_task` step.

Then call `list_site_resources { site_id: plan.site_id }` — the third-party tools this site is wired to (monitoring today; analytics/hosting later). Each entry gives `provider`, `access_method`, the non-secret connection recipe, the per-site config (`service`, `env`, `project_slug`, `app_id`, …), `mcp_server_name`, and `credential_keys` — the **names** of keys a machine must have in `~/.fabrio/credentials.json`. Fabrio never stores the values. **This is your source list: prefer a real attached resource over asking a human.**

---

## Step 2 — Check for open questions (don't plan while blocked)

If the job already has an **open question** (`is_blocked` true), stop:
> ⏸ This job is waiting on an answer. Answer it in the job's Questions tab, then re-run `$fabrio:plan-job {item_number}`.

Then load prior **decisions** so you don't re-ask what's settled: `list_decisions { site_id: plan.site_id, status: "decided" }`. A `decided` decision is binding — apply its `chosen_option_key`/`chosen_rationale`.

---

## Step 3 — Decide whether you need human input

Read the `description` and decide: **can you write a concrete, runnable plan without guessing at something a human should decide?** Real decisions worth asking about: *which* tracker/tool when the description is vague, *which* project/board/label, the top-N cap, or which department the filed tickets belong to. Do **not** ask about things you can reasonably choose or that a `decided` decision already covers.

**Resources answer the old "where do credentials come from?" question — stop asking it.** If Step 1 returned a resource whose type/provider matches what the description names, use it and record its id in the plan. Only ask when:
- the site has **no** attached resource for the category the description needs (e.g. "pull error rates" with nothing of type `monitoring` attached), or
- **two or more** attached resources of that type could plausibly serve it.

In the first case the fix is a human action in the web UI, not an answer — say so verbatim so it's actionable, and stop:
> This job needs a **monitoring** resource for {site.name}, but none is attached. Add one in **Fabrio → Resources**, attach it to this site, then re-run `$fabrio:plan-job {item_number}`.

In the second case, prefer a structured `create_decision` whose `options` are the candidate resources (`key` = resource id, `label` = "{provider} — {name}").

If you need input, open a blocking question **on the job** (use the `id` from Step 1 as `plan_item_id`) and stop:
- **Structured decision** (preferred when there are clear options): `create_decision { site_id, key, title, description, options:[…] }` (idempotent), then `create_task_question { plan_item_id: id, content, decision_id }`.
- **Freeform**: `create_task_question { plan_item_id: id, content }`.

Then stop:
> ⏸ Clarification needed to plan this job — answer in the Questions tab, then re-run `$fabrio:plan-job {item_number}`.

(Opening a question auto-blocks the job; `list_due_plan_items` skips it until answered.)

---

## Step 3.4 — The contract: steps plan, tasks do

**A job's step tree is PLANNING ONLY.** A run's permitted side effects are exactly three:

1. **read** a source, 2. **create a task**, 3. **record a receipt** (the runner does this itself).

It never edits a repo, writes a deliverable, publishes, sends, or spends. Every unit of work product is produced inside a task, by `$fabrio:execute-task`.

If the description says *"write and publish the weekly blog post"*, the tree **does not write it**. The tree decides *which* post, gathers what the writer needs, and files a task whose description contains all of it. The task writes the post and opens the PR.

**Never author a step that:** checks out a branch, edits or commits a file, opens a PR, writes a deliverable, publishes, posts, sends, or spends. If you catch yourself writing one, it belongs in the `create_task` step's `instructions` as part of the task spec — not as a step of its own.

**Receipts and cadence are not "work."** This rule is about work product, not bookkeeping — the runner records every run and advances `next_run_at` on its own. Never add steps for that.

**A run that creates zero tasks is a successful run.** If the source is empty, or everything is already covered by an open task, the job records `0 items` and stops. **Never invent work to satisfy a rule.**

---

## Step 3.5 — Settle the execution mode

`execution_mode` describes **the tasks this job files** — not the job's own steps, which always end at `create_task`. The filed tasks inherit it.

If `item.execution_mode` from Step 1 is already set, honour it. If it's null, decide from the `description` — **where does the output of each filed task end up?**

- **`repo`** — the filed task ships files in a site repo (branch → PR). A weekly blog post committed to `content/posts/`, a monthly dependency bump, generated sitemap entries. *The job itself never touches the repo.*
- **`artifact`** — the filed task produces a document with no repo home: a monthly performance report, a refreshed keyword backlog, a competitor scan. Saved as markdown on the task.
- **`external`** — the filed task prepares an action on a third-party system: posting to social, sending a campaign, adjusting ad spend, outreach. It produces a ready-to-execute package; **a human performs the action, always.**

Persist it so filed tasks route correctly: `update_plan_item { plan_item_id: id, fields: { execution_mode: "{mode}" } }`. Say which mode you chose and why in the Step 6 summary.

A job whose description mixes modes ("write the post **and** promote it on social") is two initiatives, not one — file a question rather than authoring a plan that ends in an action Fabrio may not take:
`create_task_question { plan_item_id: id, content: "This job mixes producing content (a PR) with publishing it externally. Should I split it into two plan items?" }` and stop.

---

## Step 4 — Design the step tree

A job's procedure is an ordered, **nested** tree of steps, not one blob of prose. Draft that tree.

**Every tree ends in one or more `create_task` steps. Nothing else in the tree changes anything.** The mode from Step 3.5 shapes what you write *inside* that step's `instructions` — it never adds a "commit it" or "publish it" step, because those belong to the task (Step 3.4).

**What makes a step.** One step = one verb over one input. Nest a step **under** another when it must run *once per item* of that step's output — that is a `foreach`. Use a `branch` when a step only sometimes applies. Every leaf must be executable without re-reading the original description.

**Step types**
- `action` (default) — read, filter, decide one thing.
- `foreach` — repeat its `children` once per item of `foreach_source` (`"step:<output_key>"`, or omit to iterate the immediately preceding step's output). Set `max_iterations` to cap a run; the runner defaults to 10.
- `branch` — run its `children` only when `condition` holds.
- `create_task` — **the handoff.** Files one task from the current context. Put it **inside the `foreach`** when the job files one task per item found; put it **at the root** when the job files one task per run. Its `instructions` are the **task spec**: title format, everything the executor needs in the description, what "done" looks like, and difficulty if it varies. It must be a **leaf** — never nest steps under it.

`replace_job_steps` **rejects** a tree with no `create_task` step, and rejects a `create_task` step that has children.

Give a step an `output_key` (snake_case) when a later step needs to name its result.

**Put the detail in `instructions`, not in the titles.** Concrete queries, field names, path mappings, title formats, redaction rules, noise filters — all of it belongs in the owning step's `instructions`, verbatim. A tree of thin titles is a regression against a good prose plan.

**Limits:** depth 3, 12 top-level steps, 60 total. If you need more, fold detail into `instructions` rather than adding levels.

**Cover these four concerns across the tree.** They apply whether the job files ten tasks a run or one — only the *cardinality* differs, and that's decided by where you put the `create_task` step.

1. **Gather** — how to read the source(s). When the source is an attached resource, **open that step's `instructions` with a machine-readable line** so `$fabrio:run-job` can preflight it before doing any work:

   `resource: <resource_id> (<provider> · <access_method> · mcp: <mcp_server_name>)`

   Then spell out the concrete call — which MCP tool or HTTP endpoint, and the query/filter, scoped by the resource's per-site config (e.g. `service:{config.service} env:{config.env}` for Datadog, `project:{config.project_slug}` for Sentry). **Never inline a credential value** — name the `credential_keys` and let the runner read them from `~/.fabrio/credentials.json`. If the resource exists in Fabrio but nobody has connected it on a machine yet, still author the steps: `$fabrio:run-job` preflights and fails cleanly with the setup command rather than half-running.
2. **Narrow** — validate, normalize, filter, rank. May end at many (cap with a top-N), at exactly one, or at **zero** — zero is a legal, successful outcome.
3. **Dedup** — drop anything already covered by an open task from this job (`$fabrio:run-job` gets `open_tasks` from `get_plan_item`). Its own step, **before** the `create_task` step. This applies to single-output jobs too: a weekly-blog job dedups its chosen topic against posts already queued or published.
4. **Create the task(s)** — the `create_task` step. Everything the executor needs goes in its `instructions`; the executor **must never have to re-derive what the job already read**, because it cannot see the job's steps.

Do **not** add a "record the run" step — the runner records every run itself.

**Fan-out shape** (many tasks per run — bug triage, content audits):
```
1. Preflight the monitoring resource                    action        → resource
2. Fetch and cluster errors for the window              action        → clusters
3. Filter known noise                                   action        → candidates
4. Rank and cap to the top 3                            action        → selected
5. For each selected issue             foreach (step:selected, max 3)
   5.1 Investigate the codebase for the fault           action
   5.2 File a task with the remediation plan            create_task
```

**Single-output shape** (one task per run — a weekly post, a monthly report):
```
1. Fetch the next backlog topic                         action        → topic
2. Validate and normalize the topic                     action        → prepared_topic
3. Dedup against already-published and queued posts     action        → is_new
4. Create a task to write the post                      create_task
```
Step 4's `instructions` carry the whole brief — the slug, the category, the frontmatter schema, the target keyword, which internal links to use, the site's voice notes, and the acceptance criteria. **The task writes the post, commits it, and opens the PR. The job does none of that** — there is no step 5.

---

## Step 5 — Save it

Call `replace_job_steps { plan_item_id: id, steps: [ … ] }` (use the `id` from Step 1). It replaces the job's whole tree, so send every step each time.

If it **rejects** the tree, the message says exactly what to fix. The two rules that catch a tree authored the old way:
- *"A job's steps plan work; they never do it"* — you never reached a `create_task` step. Find the step that produces the output (writes the file, drafts the copy) and turn it into the task spec.
- *"create_task step … has children"* — you nested the work under the handoff. Fold those children into the step's `instructions`.

**Do not write `job_plan`.** It is generated from the steps server-side, and `update_plan_item` rejects the field once a job has any.

If `replace_job_steps` isn't available in this session, the connected Fabrio predates nested steps: fall back to `update_plan_item { plan_item_id: id, fields: { job_plan: "<the procedure as numbered prose>" } }` and tell the user to update Fabrio.

---

## Step 6 — Output summary

Derive the "files" line from where the `create_task` step sits: inside a `foreach` → up to N per run; at the root → one per run.

```
✅ Job procedure saved for "{item.title}" ({frequency} · {department} · {execution_mode})
{N} steps{, including a foreach over {what}}
Source: {how it gathers work — e.g. "Jira project ABC via MCP"}
Files:  {"up to {N} tasks/run, deduped against open ones" | "one task/run"}
Each task {repo: "ships files in the repo and opens a PR" | artifact: "produces a markdown deliverable" | external: "prepares a package — you perform the action"}.

Run it now with $fabrio:run-job {item_number} — or let the due-jobs run run it on schedule.
```
