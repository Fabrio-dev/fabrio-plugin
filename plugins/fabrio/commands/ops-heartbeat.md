---
description: "Runs one cycle of the autonomous ops loop — queues due recurring work, implements ready tasks, proposes plan revisions, consolidates learnings, and flags stale work. Never merges."
---

# Ops Heartbeat

One cycle of the self-improving loop. Both gates stay intact: **PRs always wait for your review, and nothing merges** — this skill never calls `/merge-task`.

**Invocation:** `/ops-heartbeat` (daily) or `/ops-heartbeat --weekly` (also run the weekly steps now). Trigger-agnostic: by hand, via `/loop`, or from cron/launchd/a cloud routine.

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials or curl. The server scopes everything to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**. Headless `claude -p` children spawned in Step 2 inherit this MCP connection automatically (user-scope connections are available everywhere; a local-scope one is inherited when the child runs from the same directory).

---

## Step 0 — Setup + Run-Lock

If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected, with these steps:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
> ```
> Restart Claude Code and re-invoke.

**Acquire the soft run-lock** — call `open_ops_run { force_weekly: <true if --weekly> }`.
- If it returns `{ locked: true }`, stop: `⏸  An ops run is already in progress — skipping.`
- Otherwise capture `run_id` and `weekly_due` (the server computes both the 2h lock and whether the weekly steps are due).

Keep a tally: `items_queued, tasks_implemented, prs_opened, revisions_proposed, learnings_consolidated, stale_flagged`, plus a per-tier `models_used` map (`{light,standard,heavy}`).

**Load the tier → model mapping** with `get_model_tiers` (Step 2 dispatches each task on its tier's model).

> **Run the heartbeat itself on the cheapest model.** This skill is orchestration, so the canonical scheduled invocation is `claude -p "/ops-heartbeat" --model haiku …`. Implementation quality is not bounded by the orchestrator's model, because Step 2 dispatches each code task to the model its `difficulty` resolves to.

---

## Step 1 — Queue Due Recurring Items

Call `list_due_plan_items` — it returns recurring items with `next_run_at <= now`, each already carrying `eligible` and (if not) `skip_reason`, with the plan-status, dependency, and in-flight-task gates **resolved server-side**.

For each item where `eligible === true`: call `queue_plan_item_task { item_id }`. The server creates the task **and** advances `next_run_at` by the item's cadence in one call. Increment `items_queued`. Skip items where `eligible === false` (the `skip_reason` explains why — plan not active, prerequisite not done, or a task already in flight).

`one_time` items are never returned here — those are queued from the plan UI.

---

## Step 2 — Implement Ready Code Tasks (sequentially)

Finish **as many ready tasks as possible, strictly one at a time** — never in parallel. When a task can't complete, move on.

Call `list_tasks { type: "feature_request", statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }`. Sort so `changes_needed` comes before `ready` (review feedback first). If none, skip this step. Keep a set **`blocked_this_run`**. For each task **T** in order:

**a. Dependency gate** — if T came from a plan item with a prerequisite, and that prerequisite's task is **blocked** (`is_blocked` true or its number is in `blocked_this_run`), **skip T**: add T to `blocked_this_run` and log the cascade:
`log_task_history { task_id: T.id, action: "blocked_by_dependency", notes: "Skipped: depends on task #{prereq} which is blocked — both blocked until #{prereq} is resolved." }`
(A prerequisite merely *not done yet* but not blocked does **not** skip T — only a blocked one cascades.) You can see a task's plan item + dependency via `get_task`/`list_tasks`; use `list_due_plan_items` output or `get_plan` when you need the dependency chain.

**b. Resolve the model** for T's tier (default `standard` when `difficulty` is null) from the `get_model_tiers` result.

**c. Dispatch** — run T as its own headless session on the resolved model. Launch it from the **same directory you're running this heartbeat from** — that guarantees the child inherits the `fabrio` connection whether it was added at user scope (available everywhere) or local scope (tied to this directory). The `/feature-request` command itself is always available once the plugin is installed. Headless `-p` mode can't prompt, so tools must be allow-listed (see Permissions note):
```bash
claude -p "/feature-request {T.task_number}" --model "$MODEL" --permission-mode acceptEdits
```
The child owns all DB writes (the claim, the PR, its own history + retrospective). After it exits, **re-read** T with `get_task { task_number }` and tally from that: if now `under_review` with a PR, increment `tasks_implemented` + `prs_opened` and bump `models_used[{tier}]`; if it posted a question / is blocked, add T to `blocked_this_run`.

**Fallback:** if the tier lookup is empty, `command -v claude` fails, or the child exits non-zero **without** having claimed T (status still `ready`/`changes_needed`), invoke **`/feature-request {T.task_number}`** inline in this session instead, and log it:
`log_task_history { task_id: T.id, action: "dispatch_fallback", notes: "Headless dispatch unavailable/failed — ran /feature-request inline on the current model." }`
Re-running a partially-done task is safe — Step 3.5 of `/feature-request` resumes from existing work.

**d. Continue** — move to the next task regardless of outcome; never stop early. (Opens PRs, **never merges**.)

---

## Step 3 — Weekly Steps (revisions + consolidation)

Run when `weekly_due` from Step 0 is true (or `--weekly` was passed). Otherwise skip.

1. **Propose revisions** — for each **active** plan with tasks reaching `done` since its last accepted revision, invoke **`/revise-plan {plan_number}`** (writes a *proposed* revision — never changes the live plan). Use `list_tasks { statuses: ["done"], updated_since }` and `get_plan` to find candidates. Increment `revisions_proposed` per plan.
2. **Consolidate learnings** — invoke **`/consolidate-learnings`**; set `learnings_consolidated=true`.

---

## Step 4 — Flag Stale Work

Call `list_stale_work { older_than_days: 7 }` — it returns `stale_under_review` tasks and `stale_open_questions`. For each, log and count:
`log_task_history { task_id, action: "stale_flagged", notes: "Under review for >7 days — needs a human decision" }`
Increment `stale_flagged`. Never merges, closes, or reopens anything.

---

## Step 5 — Finish

Call `close_ops_run { run_id, status: "completed", summary: { items_queued, tasks_implemented, prs_opened, tasks_blocked, revisions_proposed, learnings_consolidated, stale_flagged, models_used: {light,standard,heavy} } }`.

Report:
```
🩺 Ops heartbeat complete.
  Recurring items queued:   {N}
  Tasks implemented (PRs):  {N}   ({light} light · {standard} standard · {heavy} heavy)
  Tasks blocked/skipped:    {N}   ← need input, or waiting on a dependency
  Plan revisions proposed:  {N}   ← review at /plans
  Learnings consolidated:   {yes|no}
  Stale items flagged:      {N}
PRs are open and waiting for your review. Nothing was merged — run /merge-task <n> when ready.
```
**On any fatal error**, release the lock: `close_ops_run { run_id, status: "failed", notes: "Failed at step {N}: {error}" }`.

---

## Permissions — required for unattended (headless) dispatch

Step 2 dispatches each task with `claude -p … --permission-mode acceptEdits`. In `-p` mode Claude Code **cannot prompt** — any tool not on the allow-list fails silently (surfacing as a task that didn't get worked, then falls back to inline). For unattended runs, ensure `.claude/settings.json` allow-lists `mcp__fabrio` (all its tools) plus the shell tools `/feature-request` uses: `gh`, `git`, `npm run`, `npx tsc`. Generate a tuned allow-list from real runs with `/fewer-permission-prompts`. Do **not** use `--dangerously-skip-permissions` — an explicit allow-list is safer and sufficient.
