---
description: "Runs every recurring job that is due — walks each job's steps to file the tasks it needs, auto-plans jobs that have no procedure yet, proposes plan revisions weekly, consolidates learnings, and flags stale work. Files tasks; never executes them."
---

# Run Due Jobs

One cycle of the **recurring** side of the self-improving loop: run the jobs that are due, plan the ones that still need a procedure, and do the weekly upkeep. A job **plans**; the tasks it files are what **do** the work — so this skill's permitted side effects are exactly three: read a source, create a task, record a receipt. It never implements a task, never merges, and never performs an outward action.

> **Executing tasks is a different loop.** `fabrio-runner` (the npm package) polls the task
> queue and spawns one process per ready task, scoped by that task's agent profile. This skill
> does not duplicate it — if you are looking for "why isn't my task being worked on", that is
> the runner, not this. **Run both:** the runner continuously, and this skill **daily**, or
> recurring jobs never fire and the weekly steps never run.

**Invocation:** `/fabrio:run-due-jobs` (daily) or `/fabrio:run-due-jobs --weekly` (also run the weekly steps now). Trigger-agnostic: by hand, via `/loop`, or from cron/launchd/a cloud routine.

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials or curl. The server scopes everything to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**. Headless `claude -p` children spawned in Step 1 inherit this MCP connection automatically (user-scope connections are available everywhere; a local-scope one is inherited when the child runs from the same directory).

---

## Step 0 — Setup + Run-Lock

If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected, with these steps:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
> ```
> Restart Claude Code and re-invoke.

**Acquire the soft run-lock** — call `open_ops_run { kind: "due-jobs", force_weekly: <true if --weekly> }`.
- If it returns `{ locked: true }`, stop: `⏸  A due-jobs run is already in progress — skipping.`
- Otherwise capture `run_id` and `weekly_due` (the server computes both the 2h lock and whether the weekly steps are due).

The lock is scoped to `kind`, so it holds off a second *due-jobs* run without ever blocking `fabrio-runner`'s task dispatch. Pass the `kind` — omitting it takes the legacy `heartbeat` lock instead.

Keep a tally: `jobs_run, jobs_planned, items_queued, revisions_proposed, learnings_consolidated, stale_flagged`.

> **Run this skill itself on the cheapest model.** It is orchestration, so the canonical scheduled invocation is `claude -p "/fabrio:run-due-jobs" --model haiku …`. Job quality is not bounded by the orchestrator's model — each job is dispatched to the model its own `difficulty` tier resolves to.

---

## Step 1 — Run Due Recurring Jobs

Call `list_due_plan_items` — it returns recurring jobs with `next_run_at <= now`, each already carrying `eligible` and (if not) `skip_reason`, with the plan-status, dependency, blocked, and job-plan gates **resolved server-side**. Each item carries `item_number`, `id`, `is_blocked`, `job_plan`, `description`, `difficulty`, and `needs_replan`.

**Load the tier → model mapping** with `get_model_tiers` so each job runs on its own tier's model (default `standard` when `difficulty` is null).

For each item where `eligible === true`, there is **one path**: **dispatch `/fabrio:run-job {item.item_number} --headless`** headlessly — run it from this directory so it inherits the `fabrio` connection, on the model its `difficulty` tier resolves to, with `--headless` like every other dispatch here:
```bash
claude -p "/fabrio:run-job {item.item_number} --headless" --model "$MODEL" --permission-mode acceptEdits
```
The skill walks the job's step tree, reads its sources, dedups against open tasks, files a task at each `create_task` step, records what each step did, and calls `record_generator_run` (which advances `next_run_at`). A failed run records which step stopped it, so the Runs tab shows where it got to. After it exits, increment `jobs_run` and add the run's `tasks_created` to `items_queued`.

- If headless dispatch is unavailable, record the failure (`record_generator_run { status: "failed", … }` names the step that stopped it) and move to the next item. Running a job inline is acceptable here **only** because a job's permitted side effects are read-a-source / create-a-task / record-a-receipt — it produces no code and no deliverable, so the model it runs on does not change what ships.

**A run that files zero tasks is a successful run.** Do not invent work to justify the cycle.

**Never call `queue_plan_item_task` here.** A recurring job's tasks come from its steps. That tool is for one-off initiatives (queued from the plan UI) and for `/fabrio:run-job`'s own legacy fallback — which the skill handles itself, including the `advance_cadence: false` that keeps the cadence from double-advancing.

**Auto-plan jobs that need a procedure.** For items where `description` is non-empty **and** either `skip_reason === "job plan not generated"` **or** `needs_replan === true`, **dispatch `/fabrio:plan-job {item.item_number} --headless`** headlessly (same machinery). `needs_replan` means the job's steps predate the plan/execute split — they describe doing the work rather than filing a task, so `/fabrio:run-job` refuses to walk them and falls back to queueing. Replanning is what converts it. If `/fabrio:plan-job` can finish without input it authors the steps; if it needs a human it opens a blocking question (the job becomes `is_blocked`). Either way the job still ran (or was skipped) this cycle on its own terms — the next cycle picks up the new procedure. Increment `jobs_planned`.

Skip the rest where `eligible === false` — the `skip_reason` explains why: plan not active, prerequisite not done, **blocked on a question** (a human must answer in the job's Questions tab), or `job plan not generated` with no description yet (needs a human to describe the job).

`one_time` items are never returned here — those are queued from the plan UI.

> **No git-provider check runs here.** A job never touches a repo, so it needs no git host. The
> check that used to live at this point existed only to pre-filter *repo tasks* for the
> task-execution steps, which now belong to `fabrio-runner` (whose `getDispatchQueue` does the
> same filtering server-side). Do not reintroduce it.

---

## Step 2 — Weekly Steps (revisions + consolidation)

Run when `weekly_due` from Step 0 is true (or `--weekly` was passed). Otherwise skip.

1. **Propose revisions** — for each **active** plan with tasks reaching `done` since its last accepted revision, invoke **`/fabrio:revise-plan {plan_number}`** (writes a *proposed* revision — never changes the live plan). Use `list_tasks { statuses: ["done"], updated_since }` and `get_plan` to find candidates. Increment `revisions_proposed` per plan.
2. **Consolidate learnings** — invoke **`/fabrio:consolidate-learnings`**; set `learnings_consolidated=true`.

---

## Step 3 — Flag Stale Work

Call `list_stale_work { older_than_days: 7 }` — it returns `stale_under_review` tasks and `stale_open_questions`. For each, log and count:
`log_task_history { task_id, action: "stale_flagged", notes: "Under review for >7 days — needs a human decision" }`
Increment `stale_flagged`. Never merges, closes, or reopens anything.

---

## Step 4 — Finish

Call `close_ops_run { run_id, status: "completed", summary: { jobs_run, jobs_planned, items_queued, revisions_proposed, learnings_consolidated, stale_flagged } }`.

Report:
```
🔁 Due jobs complete.
  Jobs run:                 {jobs_run}   → {items_queued} tasks filed
  Job plans generated:      {N}   ← jobs auto-planned or replanned this run
  Plan revisions proposed:  {N}   ← review at /plans
  Learnings consolidated:   {yes|no}
  Stale items flagged:      {N}
Tasks filed this cycle are picked up by fabrio-runner (or /fabrio:execute-task by hand).
Nothing was merged, published or sent.
```
Omit any line that is zero, so a quiet cycle still reads cleanly.
**On any fatal error**, release the lock: `close_ops_run { run_id, status: "failed", notes: "Failed at step {N}: {error}" }`.

---

## Permissions — required for unattended (headless) dispatch

Step 1 dispatches `/fabrio:run-job` and `/fabrio:plan-job` with `claude -p … --permission-mode acceptEdits`. Headless `-p` children need two things, or they fail silently (surfacing as a job that never ran):

1. **An authenticated, persistent CLI session.** A scheduled/cron run has no one to sign in mid-run, so `claude auth login`'s session must be valid — or, better for unattended use, set a long-lived `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) or `ANTHROPIC_API_KEY` in `~/.claude/settings.json` `env`. If auth is missing, every child dies at startup and nothing runs.
2. **A user-scope tool allow-list.** In `-p` mode Claude Code cannot prompt, so any tool not allow-listed fails. Put the list in **`~/.claude/settings.json`** (user scope, so it applies wherever a child runs): `mcp__fabrio` (all its tools) plus `Bash(claude:*)`. A job's own steps may also need whatever its **resource** requires — typically another MCP server; see **Fabrio → Settings → Tools** for the authoritative set. A tool a job needs but the machine does not permit fails at the moment it is used, not at startup, so it surfaces as a run that got part-way and stopped.

Both are walked through in **Fabrio → Settings → API keys**. Generate a tuned allow-list from real runs with `/fewer-permission-prompts`. Do **not** use `--dangerously-skip-permissions` — an explicit allow-list is safer and sufficient.
