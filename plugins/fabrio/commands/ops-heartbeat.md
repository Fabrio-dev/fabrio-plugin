---
description: "Runs one cycle of the autonomous ops loop — queues due recurring work, executes ready tasks in every department, proposes plan revisions, consolidates learnings, and flags stale work. Never merges and never performs an outward action."
---

# Ops Heartbeat

One cycle of the self-improving loop, across **every department** — development, design, marketing and content all execute here. The gates stay intact: **PRs always wait for your review, nothing merges** (this skill never calls `/fabrio:merge-task`), and **no outward action is ever performed** — work that needs publishing, sending or spending is prepared for you and stops there.

**Invocation:** `/fabrio:ops-heartbeat` (daily) or `/fabrio:ops-heartbeat --weekly` (also run the weekly steps now). Add `--chain` to have Step 2 handle ready **repo** tasks by **auto-grouping** them into dependency chains via `/fabrio:feature-chain` (dependent tasks build on one shared branch and land as a single PR) instead of the default one-PR-per-task dispatch; non-repo tasks still run individually. Flags combine (`--weekly --chain`). Trigger-agnostic: by hand, via `/loop`, or from cron/launchd/a cloud routine — `--chain` is meant for the unattended scheduled runs where grouping dependent work into one PR is worth more than per-task model routing.

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

Keep a tally: `items_queued, jobs_planned, generators_run, tasks_generated, tasks_implemented, prs_opened, revisions_proposed, learnings_consolidated, stale_flagged`, plus a per-tier `models_used` map (`{light,standard,heavy}`).

**Load the tier → model mapping** with `get_model_tiers` (Step 2 dispatches each task on its tier's model).

> **Run the heartbeat itself on the cheapest model.** This skill is orchestration, so the canonical scheduled invocation is `claude -p "/fabrio:ops-heartbeat" --model haiku …`. Implementation quality is not bounded by the orchestrator's model, because Step 2 dispatches each code task to the model its `difficulty` resolves to.

---

## Step 1 — Run Due Recurring Jobs

Call `list_due_plan_items` — it returns recurring jobs with `next_run_at <= now`, each already carrying `eligible` and (if not) `skip_reason`, with the plan-status, dependency, blocked, and job-plan gates **resolved server-side**. Each item carries `item_number`, `id`, `kind` (`execution` | `generator`), `is_blocked`, `job_plan`, and `description`.

For each item where `eligible === true`, branch on `kind`:

- **`execution`** (or missing) → call `queue_plan_item_task { item_id: item.id }`. The server creates a task for **each site the item targets** (one for a single-site item; one per site for a multi-site or all-sites plan, skipping sites already covered) **and** advances `next_run_at` in one call. It returns `tasks` (an array) — increment `items_queued` by `tasks.length`.
- **`generator`** → **dispatch `/fabrio:run-generator {item.item_number}`** headlessly (same machinery as Step 2 — run it from this directory so it inherits the `fabrio` connection; resolve the model from the item's `difficulty` tier). The skill walks the job's step tree, reads the source, dedups against open tickets, files N tasks, records what each step did, and calls `record_generator_run` (which advances `next_run_at`). A failed run records which step stopped it, so the Runs tab shows where it got to. After it exits, increment `generators_run` and add the run's `tasks_created` to `tasks_generated`. **Do not** also call `queue_plan_item_task` for a generator.
  - Fallback: if headless dispatch is unavailable, run `/fabrio:run-generator {item.item_number}` inline on the current model.

**Auto-plan jobs that need a procedure.** For items where `skip_reason === "job plan not generated"` **and** `description` is non-empty, **dispatch `/fabrio:plan-job {item.item_number}`** headlessly (same machinery). If `/fabrio:plan-job` can finish without input it authors the job's steps; if it needs a human it opens a blocking question (the job becomes `is_blocked`) — either way, don't run the job this cycle; the next heartbeat picks it up once it has a procedure and isn't blocked. Increment `jobs_planned`.

Skip the rest where `eligible === false` — the `skip_reason` explains why: plan not active, prerequisite not done, **blocked on a question** (a human must answer in the job's Questions tab), a task already in flight for an execution job, or `job plan not generated` with no description yet (needs a human to describe the job).

`one_time` items are never returned here — those are queued from the plan UI.

---

## Step 2 — Execute Ready Tasks (sequentially)

Finish **as many ready tasks as possible, strictly one at a time** — never in parallel. When a task can't complete, move on.

**Every department's work is in scope**, not just code. `/fabrio:execute-task` classifies each task's `execution_mode` and routes it: `repo` work goes on to `/fabrio:feature-request` (branch → PR), `artifact` work produces a reviewable markdown deliverable, and `external` work produces a ready-to-execute package for a human. The heartbeat's ceiling is unchanged and applies to all three: **it opens PRs and prepares packages, but never merges and never performs an outward action.**

> **Mode.** Default (no `--chain`): execute each task individually via `/fabrio:execute-task` — steps **2a–2d** below. With **`--chain`**: additionally auto-group dependent **repo** tasks into shared-branch chains via `/fabrio:feature-chain` — see **Step 2C**, which runs *before* 2a–2d and takes the repo tasks out of the per-task pass.

Call `list_tasks { statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }` — no `type` or `execution_mode` filter, so nothing is silently left behind. Sort so `changes_needed` comes before `ready` (review feedback first). If none, skip this step. Keep a set **`blocked_this_run`**. For each task **T** in order:

**a. Dependency gate** — if T came from a plan item with a prerequisite, and that prerequisite's task is **blocked** (`is_blocked` true or its number is in `blocked_this_run`), **skip T**: add T to `blocked_this_run` and log the cascade:
`log_task_history { task_id: T.id, action: "blocked_by_dependency", notes: "Skipped: depends on task #{prereq} which is blocked — both blocked until #{prereq} is resolved." }`
(A prerequisite merely *not done yet* but not blocked does **not** skip T — only a blocked one cascades.) You can see a task's plan item + dependency via `get_task`/`list_tasks`; use `list_due_plan_items` output or `get_plan` when you need the dependency chain.

**b. Resolve the model** for T's tier (default `standard` when `difficulty` is null) from the `get_model_tiers` result.

**c. Dispatch** — run T as its own headless session on the resolved model. Launch it from the **same directory you're running this heartbeat from** — that guarantees the child inherits the `fabrio` connection whether it was added at user scope (available everywhere) or local scope (tied to this directory). The `/fabrio:execute-task` command itself is always available once the plugin is installed. Headless `-p` mode can't prompt, so tools must be allow-listed (see Permissions note):
```bash
claude -p "/fabrio:execute-task {T.task_number}" --model "$MODEL" --permission-mode acceptEdits
```
The child owns all DB writes (the claim, the mode classification, the PR or deliverable, its own history + retrospective). After it exits, **re-read** T with `get_task { task_number }` and tally from that:

- now `under_review` **with a `pr_number`** → `tasks_implemented`++, `prs_opened`++, bump `models_used[{tier}]`
- now `under_review` **with a `deliverable`** and no PR → `tasks_implemented`++, `deliverables_produced`++, bump `models_used[{tier}]`; if its `execution_mode` is `external`, also `awaiting_human_action`++ (these need a person before they can be approved, so surface them separately)
- posted a question / `is_blocked` → add T to `blocked_this_run`

**Fallback:** if the tier lookup is empty, `command -v claude` fails, or the child exits non-zero **without** having claimed T (status still `ready`/`changes_needed`), invoke **`/fabrio:execute-task {T.task_number}`** inline in this session instead, and log it:
`log_task_history { task_id: T.id, action: "dispatch_fallback", notes: "Headless dispatch unavailable/failed — ran /fabrio:execute-task inline on the current model." }`
Re-running a partially-done task is safe — Step 3.5 of `/fabrio:execute-task` resumes from existing work.

**d. Continue** — move to the next task regardless of outcome; never stop early. (Opens PRs and prepares external packages; **never merges, never publishes or sends anything**.)

---

## Step 2C — Implement Repo Tasks via Chains (only when `--chain`)

Runs **before** 2a–2d and covers **only `execution_mode: "repo"` tasks** — a chain is a shared git branch, so it is meaningless for a deliverable or an external action. `/fabrio:feature-chain` owns that repo pass: it groups the workable repo tasks into dependency chains, builds each chain on one shared branch, holds a chain (no PR) if a task needs input, and opens one PR per chain. So ops-heartbeat does **not** iterate those tasks or run the per-task dependency gate for them — `feature-chain` handles ordering and cascades internally. It also **routes models per task itself** (Step 2.5 of that skill): whenever headless dispatch is available it dispatches each task on its own tier's model, regardless of the orchestrator's model. So you don't size the model here — let `feature-chain` orchestrate on a cheap model and fan out per task.

**After this step, run 2a–2d for the remaining tasks** — everything from the Step 2 list whose `execution_mode` is `artifact`, `external`, or **not yet set** (an unclassified task might turn out to be repo work, but only `/fabrio:execute-task` can decide that, and it will be picked up by the next heartbeat's chain pass once classified). Skip any task the chain pass already advanced.

From the Step 2 `list_tasks` result, filter to `execution_mode == "repo"` and take the distinct `site_id`s that have such tasks. If there are none, skip straight to 2a–2d. For **each such site**, in order:

**a. Model** — dispatch the `feature-chain` orchestrator on the **cheapest** tier's model (like the heartbeat itself — it only does git, grouping, and PR bookkeeping; the real implementation runs in the per-task children it spawns, each on its own mapped model). Load the map with `get_model_tiers` if you haven't.

**b. Dispatch** — run `/fabrio:feature-chain` in auto-group mode scoped to that site, headless on that cheap model, from **this directory** (so it inherits the `fabrio` connection):
```bash
claude -p "/fabrio:feature-chain --site {site_id}" --model "$ORCH_MODEL" --permission-mode acceptEdits
```
The child (and its own per-task grandchildren) own all DB writes — claims, chain branches, the PR(s), history, retrospectives, and per-task model routing. Per-site scoping keeps each site's chains independent.

**c. Tally** — after the child exits, re-read that site's tasks with `list_tasks { site_id, execution_mode: "repo", statuses: ["under_review","in_progress","changes_needed"] }` (or the original numbers via `get_task`): each task now `under_review` **with a `pr_number`** → increment `tasks_implemented` and bump `models_used[{tier}]`; count each **distinct new `pr_number`** once toward `prs_opened` (a chain of N tasks is one PR); any task left `in_progress` with no PR or `is_blocked` (a held chain) → add to `blocked_this_run`.

**d. Fallback** — if the tier lookup is empty, `command -v claude` fails, or the child exits non-zero without any of that site's tasks advancing, invoke `/fabrio:feature-chain --site {site_id}` **inline** in this session and log it on one representative task:
`log_task_history { task_id, action: "dispatch_fallback", notes: "Headless chain dispatch unavailable/failed — ran /fabrio:feature-chain --site {site_id} inline." }`
Re-running is safe — `feature-chain` skips work already committed on a chain branch.

**e. Continue** to the next site regardless of outcome, then fall through to **2a–2d** for the non-repo and unclassified tasks. (Opens PRs, **never merges**.)

---

## Step 3 — Weekly Steps (revisions + consolidation)

Run when `weekly_due` from Step 0 is true (or `--weekly` was passed). Otherwise skip.

1. **Propose revisions** — for each **active** plan with tasks reaching `done` since its last accepted revision, invoke **`/fabrio:revise-plan {plan_number}`** (writes a *proposed* revision — never changes the live plan). Use `list_tasks { statuses: ["done"], updated_since }` and `get_plan` to find candidates. Increment `revisions_proposed` per plan.
2. **Consolidate learnings** — invoke **`/fabrio:consolidate-learnings`**; set `learnings_consolidated=true`.

---

## Step 4 — Flag Stale Work

Call `list_stale_work { older_than_days: 7 }` — it returns `stale_under_review` tasks and `stale_open_questions`. For each, log and count:
`log_task_history { task_id, action: "stale_flagged", notes: "Under review for >7 days — needs a human decision" }`
Increment `stale_flagged`. Never merges, closes, or reopens anything.

---

## Step 5 — Finish

Call `close_ops_run { run_id, status: "completed", summary: { items_queued, jobs_planned, generators_run, tasks_generated, tasks_implemented, prs_opened, deliverables_produced, awaiting_human_action, tasks_blocked, revisions_proposed, learnings_consolidated, stale_flagged, models_used: {light,standard,heavy} } }`.

`tasks_implemented` counts everything that reached `under_review` this run, whatever the mode; `prs_opened` and `deliverables_produced` split that total by how the work landed, so a cycle that shipped only content still reports real output instead of zeros. `awaiting_human_action` is the subset of `external` tasks now waiting on a person to perform them — call it out in the closing message, since those don't move without the human.

Report:
```
🩺 Ops heartbeat complete.
  Recurring items queued:   {N}
  Job plans generated:      {N}   ← jobs auto-planned this run
  Generators run:           {N}   → {tasks_generated} tickets filed
  Tasks executed:           {N}   ({light} light · {standard} standard · {heavy} heavy)
    → PRs opened:           {N}
    → Deliverables written: {N}
    → Awaiting your action: {N}   ← external work prepared, needs you to perform it
  Tasks blocked/skipped:    {N}   ← need input, or waiting on a dependency
  Plan revisions proposed:  {N}   ← review at /plans
  Learnings consolidated:   {yes|no}
  Stale items flagged:      {N}
PRs and deliverables are waiting for your review. Nothing was merged, published or sent —
run /fabrio:merge-task <n> once you've reviewed (and performed any external step).
```
Omit the three indented sub-lines that are zero, so a pure-code cycle still reads cleanly.
**On any fatal error**, release the lock: `close_ops_run { run_id, status: "failed", notes: "Failed at step {N}: {error}" }`.

---

## Permissions — required for unattended (headless) dispatch

Step 2 dispatches each task with `claude -p … --permission-mode acceptEdits` (`/fabrio:execute-task` by default — which itself dispatches `/fabrio:feature-request` for repo work — or `/fabrio:feature-chain --site …` under `--chain`, which dispatches per-task `--step` grandchildren). Headless `-p` children need two things, or they fail silently (surfacing as a task that didn't get worked, then falls back to inline):

1. **An authenticated, persistent CLI session.** A scheduled/cron heartbeat has no one to sign in mid-run, so `claude auth login`'s session must be valid — or, better for unattended use, set a long-lived `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) or `ANTHROPIC_API_KEY` in `~/.claude/settings.json` `env`. If auth is missing, every child dies at startup and nothing gets worked.
2. **A user-scope tool allow-list.** In `-p` mode Claude Code cannot prompt, so any tool not allow-listed fails. Put the list in **`~/.claude/settings.json`** (user scope — children run in each site's repo dir, where a project `.claude/settings.json` wouldn't apply): `mcp__fabrio` (all its tools) plus `Bash(git:*)`, `Bash(gh:*)`, `Bash(npm run:*)`, `Bash(npx:*)`, `Bash(claude:*)`.

Both are walked through in **Fabrio → Settings → API keys**. Generate a tuned allow-list from real runs with `/fewer-permission-prompts`. Do **not** use `--dangerously-skip-permissions` — an explicit allow-list is safer and sufficient.
