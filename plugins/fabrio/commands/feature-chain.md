---
description: "Implements a set of dependent feature tasks on one shared branch — each builds on the last — and opens a single PR for the whole chain. Auto-groups available tasks into chains, or takes an explicit ordered list."
---

# Feature Chain

Implement **dependent** `execution_mode='repo'` tasks together: run them one-at-a-time on a **single shared branch** (each task builds on the previous task's commits), then open **one PR** covering the whole chain. This is for work that was split into several sequential tasks where you don't want to merge task 1 into the base branch just to start task 2 — build and test them together, check them in once.

- `/fabrio:feature-chain 12 13 14 15` — **explicit chain.** The order you give **is** the build order. All must be on the same site.
- `/fabrio:feature-chain --site <site_id|name>` — **auto-group** the available tasks for one site into dependency chains and run each.
- `/fabrio:feature-chain` — **auto-group across all sites** (one or more chains per site).

**Contrast with `/fabrio:feature-request`:** that skill gives every task its own branch and its own PR. Use it for independent tasks. Use **this** skill when tasks build on each other. A chain of one task produces exactly the same result as `feature-request` (one branch, one PR) — so auto-group mode safely handles standalone tasks too.

**Hold on block:** if any task in a chain needs human input (open question / a decision), the chain **stops there** and opens **no PR** — the completed work stays on the shared branch, and re-running resumes once the block is resolved.

**Resume:** re-running detects work already committed on the shared branch (per-task `Task #{n}:` commit markers) and picks up from the first task that isn't done yet.

**Model routing (automatic):** whenever headless dispatch is available, every task is implemented in its own child on its tier's mapped model (all on the shared branch), so the chain isn't stuck on this session's model. It falls back to inline on the current model only when dispatch is unavailable, with a warning. See Step 2.5.

---

## Step 0 — Setup

All Fabrio data access goes through the **`fabrio` MCP server** (tools named `mcp__fabrio__*`). There are **no** Supabase credentials or curl helpers — the server authenticates with a per-account API key and returns only the active account's data.

Run first; if any check fails, stop:

**Workspace git provider — do this before anything else, no default, ever.** Call `get_account_context`. **If `git_provider` is null, stop the entire run** (a chain can't start without a git host to open its PR against). Do not group tasks, create a branch, or edit a file first. Print exactly:
> `Error: No git provider is selected for this workspace. Set it in Fabrio → Settings → AI instructions, then re-run /fabrio:feature-chain.`

If `git_provider` is set, run its `ops.auth_check`. On failure, stop and print `git_provider.auth_hint` verbatim. **Never fall back to another provider, and never guess one from the git remote.**

Store the resolved provider as `PROVIDER` — every `PROVIDER.ops.*` reference in the rest of this skill means "run that command, substituting placeholders." `{repo}`/`{org}`/`{project}` come from `PROVIDER.coordinates` applied to the current repo's git remote; for GitHub these are inferred automatically by `gh` from the working directory. `get_task` (Step 3b) returns the same resolved provider as `task.account.git_provider`.

**A `PROVIDER.ops.*` failure anywhere later in the run is an infrastructure blocker, not something to route around.** The auth_check above only rules out the common case; a command can still fail mid-chain (e.g. `az repos`/`az devops` demanding an interactive login when reading PR comments) or an MCP tool call can be denied because it isn't in the dispatched task's `--allowedTools` scope. Either is a stop condition: `log_task_history { task_id: T.id, action: "error", notes: "{what failed} — {the exact remediation command or config change}" }`, leave git clean (per the mid-chain failure handling below), and report. **Never** improvise around it by calling an MCP tool outside the granted scope as a fallback — a denial is the boundary, not a suggestion to find another door. **If HEADLESS**, never raise it conversationally: there is no one to answer, so an unanswered question in chat is functionally identical to silence, except the process then exits 0 and looks "finished" to the dispatcher instead of recording what actually happened. **If not HEADLESS**, asking in chat is fine — same as Step 3e's clarification contract.

- If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected:
  > Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
  > ```
  > claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
  > ```
  > Restart Claude Code and re-invoke.

**Source root** — the site repos live at `{source_root}/{site.relative_path}` (`get_site` / `get_task` return `relative_path`). Read `source_root` from the `FABRIO_SOURCE_ROOT` environment variable. **If it's unset**, tell the user it needs to be set and give them these instructions, then ask for the path to use for this run (so the run isn't blocked):

> `FABRIO_SOURCE_ROOT` isn't set — I need it to find your repos on disk. Claude Code reads it from its own environment (not Fabrio's `.env.local`). Set it once, either way:
> - **Recommended** — add an `env` block to `~/.claude/settings.json`, then restart Claude Code:
>   ```json
>   { "env": { "FABRIO_SOURCE_ROOT": "C:\\Users\\you\\Source" } }
>   ```
>   (macOS/Linux: `"/Users/you/Source"`.)
> - **Or** set a real OS/shell env var before launching `claude`.
>
> For now, what's the absolute path to the folder that holds your site repos? I'll use it for this run.

`BASE_BRANCH` is resolved **per repo** in Step 3 (never assume `main`).

Task history: use `log_task_history` for semantic milestones. Routine field edits (status, task_plan, pr_url, …) are logged automatically by `update_task`, so don't double-log those.

---

## Step 1 — Determine Chains

Produce an ordered list of **chains**, where each chain is an ordered list of tasks that all belong to **one site** (one branch + one PR = one repo).

### Explicit mode (task numbers given)
The arguments are one chain, **in the order given** = the build order. `get_task` each one and confirm they share a `site_id`. If any belong to a different site, **stop**: `Error: chain tasks span multiple sites (#{a} → {siteA}, #{b} → {siteB}). A chain must be one repo. Run them as separate chains.` Do not reorder an explicit chain — the user's order is authoritative.

### Auto-group mode (`--site` or bare)
Fetch workable tasks with `list_tasks { execution_mode: "repo", statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }` (add `site_id` when `--site` is given; resolve a site **name** to its id via `list_sites` first). **Only `repo` tasks can be chained** — a chain *is* a shared git branch, so a deliverable or an external action has nothing to build on. Department doesn't matter: a marketing landing page and a content blog post chain like any other repo work. Non-repo and unclassified tasks belong to `/fabrio:execute-task`. If none, output "No tasks are currently available to chain." and stop.

### `--headless`
Accept this flag anywhere in the arguments (e.g. `/fabrio:feature-chain --site 3 --headless`). `/fabrio:ops-heartbeat` always passes it on its dispatch (see below); a human typing the command directly normally doesn't. Set **HEADLESS = true** for this run if the flag is present — Step 3e reads this to decide how to raise a clarification question. (A `--step {n}` child sets its own HEADLESS unconditionally — see that section — since it skips this Step entirely.)

**Group per `site_id`, then within each site cluster tasks into chains.** Chain two tasks only when there's **real evidence** one builds on the other, in priority order:

1. **Structured signal (authoritative)** — both tasks came from the same plan and their `plan_items` are linked by `depends_on_item_id`. Each task carries `plan_item_id`; call `get_plan_item { … }` (or read `get_plan`) to see the `depends_on` chain. Order the chain by that dependency; the prerequisite builds first.
2. **Semantic signal** — a task's `description`/`acceptance_criteria`/`title` references another ("after #X", "builds on the … from #X", or it clearly extends the same new table/subsystem/files an earlier task introduces). Order so the foundational task comes first.

**Bias toward NOT grouping.** When a dependency is unclear, leave the tasks as separate single-task chains (their own PRs). Wrongly coupling unrelated work into one PR is worse than not grouping. A task with no dependency is a **chain of one**.

**Print the plan before executing** — list each chain, its site, its ordered tasks, and a one-line reason for each dependency edge, e.g.:
```
Chains to run:
  • FitPlan  chain: #12 → #13 → #14   (#13 extends the workouts table #12 adds; #14 adds UI over #13's API)
  • FitPlan  single: #20             (no dependency — its own PR)
  • Ledger   chain: #31 → #32        (plan-item depends_on)
```

Then run **Steps 2–9 for each chain in order** (Step 2 onward is per-chain). In auto/multi-chain mode, a chain that holds or errors does **not** stop the others — continue to the next chain.

---

## Step 2 — Per-Chain Setup

Let `chain` = the ordered tasks. `minN` = the lowest `task_number` in the chain. Resolve the site once (all tasks share it): repo path = `{source_root}/{site.relative_path}`.

**Base branch** (from inside the repo):
```bash
cd {site_path}
BASE_BRANCH=$({PROVIDER.ops.default_branch})   # Azure DevOps: strip the "refs/heads/" prefix
```

**Branch naming comes from the workspace's `ai_context` when it specifies a convention** (see Step 3b) — the pattern below is the default, not a mandate.

**Shared branch — anchored on `minN` so it's reconstructable on resume:** `feature/chain-{minN}-{short-slug}` where `{short-slug}` is a 3–5 word kebab-case theme of the chain.

```bash
git fetch origin
git branch -a | grep "feature/chain-{minN}-"     # existing chain branch? → resume onto it
```
- **If it exists** (local or `remotes/origin/`): check it out and pull (`git checkout {branch} && git pull origin {branch}` — for a remote-only branch, `git checkout -b {branch} origin/{branch}`). Go to **Step 3 in resume mode**.
- **If not:** ensure a clean tree first (if dirty with unrelated work, `git stash push -u -m "pre-chain-{minN} WIP"` and tell the user), then:
  ```bash
  git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
  git checkout -b feature/chain-{minN}-{short-slug}
  ```

**Chain-level resume — is this chain already complete?** If any task in the chain has `pr_url`/`pr_number` set, the chain PR already exists. Treat it like `feature-request`'s "PR exists" path: check the shared PR for new human review comments newer than the last branch commit —
```bash
{PROVIDER.ops.pr_comments}   # substitute {pr_number} and {repo}/{org}/{project}; flatten Azure DevOps' threads to one chronological list
git log origin/{branch} -1 --format="%aI"
```
If there's newer feedback, read it (it applies to whichever task(s) it names), re-implement on the branch, run Step 8's build gate, push, and stop. If not, output `⏭  Chain feature/chain-{minN}-… already complete (PR #{pr_number}). Skipping.` and move to the next chain.

---

## Step 2.5 — Choose Execution Mode (MANDATORY — decide before touching any task)

**Do this before you fetch, plan, claim, implement, or commit a single task.** Deciding the mode is a hard gate — you may not enter Step 3 or Step 3R until it is done and printed. A session can't switch models mid-run, so a task only lands on its mapped model in its **own process**; this session's model is irrelevant. Implementing even one task first (as "inline") and then deciding is a **defect** that strands the whole chain on the wrong model.

1. **Ensure every task has a `difficulty`** (you need the tier to pick a model). Classify any null ones now (`light` / `standard` / `heavy` — rubric in 3f) and persist with `update_task`.
2. **Load the tier → model map:** `get_model_tiers`.
3. **Probe headless dispatch — actually test it, don't assume.** A dispatched child must (a) be able to run at all, which means the `claude` CLI is authenticated, and (b) reach the `fabrio` MCP. Run a trivial child and check it returns cleanly:
   ```bash
   claude -p "reply with exactly: ok" --model haiku 2>&1
   ```
   Headless dispatch is available **only if that prints `ok`** (exit 0). If it prints anything else, dispatch is **not** available — read the reason and fall back to inline (Step 3), telling the user the exact fix:
   - `Failed to authenticate` / `OAuth session expired` / `loggedIn: false` / any auth error → the `claude` CLI isn't signed in, so no child can run. **Tell the user:** run `claude auth login` (interactive), or for hands-off/scheduled runs `claude setup-token` and set `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`) in `~/.claude/settings.json` — see **Fabrio → Settings → API keys** for the full setup.
   - a permission/prompt error or a hang on an `mcp__fabrio` call → the `-p` child isn't allowed to use the MCP unattended. **Tell the user:** add `mcp__fabrio` (plus `Bash(git:*)`, `Bash(gh:*)` (or `Bash(az:*)` on an Azure DevOps workspace), `Bash(npm run:*)`, `Bash(npx:*)`) to `permissions.allow` in `~/.claude/settings.json` (user scope, so it applies in every repo).
   - `command not found` → `claude` not on `PATH`.

   **Never silently pretend to route.** If the probe fails, you run inline **and** print the reason + fix, so the user can enable routing.

Then pick the mode:

- **Routed mode — the default whenever the probe passed.** Go to **Step 3R**; **every** task is dispatched to a headless child on its own tier's model, even if all tasks share one tier (a chain of `heavy` tasks in a Haiku session must still route to the heavy model).
- **Inline fallback — only when the probe failed.** Go to **Step 3**; the whole chain runs in this session on its current model. Print a warning that names the probe failure and its fix: `⚠️  Headless dispatch unavailable ({probe reason}) — running the whole chain inline on {current model}; tasks are NOT routed to their mapped models. Fix: {the fix from above}.`

Print the decision. For routed mode, print the per-task plan, e.g. `Routing: #64→opus (heavy) · #66→sonnet (standard) · #67→opus (heavy) …`.

> **Hard rule for routed mode:** you (the orchestrator) do **not** implement or commit task code yourself — your only per-task action is the `claude -p … --step` dispatch in Step 3R. If you're about to edit a file or run `git commit` for a task in routed mode, **STOP**: dispatch instead. Falling back to inline because dispatch "seems easier" is a defect, not a shortcut. The only route from routed mode into inline work is the explicit per-task dispatch **fallback** in Step 3R (child errored without claiming/committing).

---

## Step 3 — Implement Each Task in the Chain — Inline fallback (in order)

**Enter this step only if Step 2.5 selected the inline fallback** (headless dispatch unavailable). If Step 2.5 selected routed mode, go to **Step 3R** and do not implement here. For each task **T** in the chain, in order. This mirrors `/fabrio:feature-request` Steps 2–9 **minus** its per-task branch creation and per-task PR — you are already on the shared branch and stay on it.

### 3a — Already done on this branch? (resume marker)
```bash
git log {branch} --grep "^Task #{T.task_number}:" -1
```
If a commit exists, T is already implemented on the chain → output `↩  #{T.task_number} already on the branch — skipping to next.` and continue to the next task. (`claim_task` returning `{ claimed:false, current_status:"in_progress" }` for **your own** interrupted run is expected and not a conflict.)

### 3b — Fetch + validate
`get_task { task_number: T.task_number, include_learnings: true, include_decisions: true, include_playbook: true }` → task + `account` + `site` + `questions` (full messages on OPEN threads only) + `attachments` + `agent` + `learnings` + `decisions` + `playbook`. Null → in a chain this breaks the build order; **hold the chain** (see Step 4) treating T as the blocker. Validate: `execution_mode == 'repo'` (else hold — a chain can't skip a prerequisite it has no way to build, and an unclassified task must go through `/fabrio:execute-task` first) and `status ∈ { ready, changes_needed, in_progress }` (`in_progress` only to resume). Note `account.ai_context` (the chain's single shared branch must follow the workspace's naming convention if it sets one), `T.playbook`, `site.ai_context`, and `task.agent` (034 — `instructions` binding, `skills` the craft references). Tasks in one chain may resolve to **different agents**; apply each task's own, not the first one's.

### 3c — Open questions → HOLD
If any `T.questions` has `status='open'`, the chain **holds at T** — go to Step 4. Everything after T depends on it, so don't attempt later tasks.

### 3d — Apply learnings & decisions
From 3b's `get_task` (no separate calls): `loaded_learnings` = `T.learnings` (treat as instructions: apply `code_pattern`/`preference`; check work against `pitfall`/`review_feedback`; follow `process`). `loaded_decisions` = `T.decisions` (binding — apply, don't re-ask).

### 3e — Review for clarity
Read the context layers widest-first — `account.ai_context` (workspace rules), then the department `playbook`, then `task.agent.instructions` (how this kind of work is done well), then `site.ai_context` (this repo), then T's `title`, `description`, `feature_summary`, `acceptance_criteria`, question threads, and any image `attachments` (view each `public_url`). All binding; the narrower layer wins a direct conflict. **No layer raises the autonomy ceiling.** For `changes_needed`, read the PR review comments. Ask: **can I implement this completely and correctly without a decision a human should make?** If `loaded_decisions` already covers the ambiguity, apply it and continue. **If clarification is still needed,** open it and **HOLD the chain** (Step 4):
- **(a) Structured decision** (choice between concrete options — prefer this): `create_decision { site_id: T.site_id, source_task_id: T.id, key, title, description, options:[…] }`, then `create_task_question { task_id: T.id, content, decision_id }`.
- **(b) Freeform question:** `create_task_question { task_id: T.id, content }` (auto-flags T blocked).

> **If HEADLESS** (the top-level `--headless` flag, or this is a `--step` child — always headless, see below): (a)/(b) are the *only* way to raise this — there is no one to answer a chat prompt, so record it and hold as just described. **If not HEADLESS** (a human is running this chain directly, no flag): asking in chat instead is fine — that's today's behavior and it's unchanged; posting via `create_task_question` is equally fine when you'd rather leave a record.

### 3f — Classify difficulty (if unset)
If `T.difficulty` is null, assign `light` / `standard` (default) / `heavy` and persist: `update_task { task_id: T.id, fields: { difficulty } }`.

### 3g — Plan (checkpoint)
Read the codebase, then save a per-task plan so an interrupted run has context: `update_task { task_id: T.id, fields: { task_plan: "<plan markdown>" } }` (auto-logs `plan_saved`). Cover Summary, Approach, Files, Database Changes (or "None"), Sub-Skills (including every skill in `task.agent.skills`), Learnings Applied, Agent Applied, Testing. Print it before writing code.

### 3h — Claim (concurrency guard)
`claim_task { task_id: T.id }` — atomically `ready|changes_needed → in_progress`.
- `{ claimed: true }` → continue.
- `{ claimed: false, current_status }` → `in_progress` is your own resume (proceed); `under_review`/`approved`/`done` means it was handled elsewhere — that breaks the chain's assumptions, so **hold** and tell the user T is already past implementation.

### 3i — Implement (on the shared branch)
Follow the plan; read adjacent files and match existing patterns exactly — including the repo's own `CLAUDE.md`/`AGENTS.md` and its DB-migration workflow. Type-check with the repo's own command as you go. **Sub-skills — invoke when applicable:** `/frontend-design`, `/react-best-practices`, `/web-design-guidelines`, `/composition-patterns`, `/ux-review`.

### 3j — Commit with the resume marker
Commit T's work in logical units; the **first line of at least one commit for T must start** `Task #{T.task_number}:` — this is the resume marker Step 3a greps for:
```bash
git commit -m "Task #{T.task_number}: {short description of what this task added}"
```
Do **not** create a PR and do **not** set `under_review` yet — T stays `in_progress` while the chain is in flight. `log_task_history { task_id: T.id, action: "chain_committed", notes: "Committed on {branch} as part of chain (min #{minN})." }`.

### 3k — Build gate before the next layer
Run `npm run build`. The next task builds on top of T, so the chain is only as sound as each layer — **do not move to the next task until the build is green.** Fix issues, re-run `npx tsc --noEmit` then `npm run build`, commit the fix (`Task #{T.task_number}: fix build`), `log_task_history { action: "build_fixed" }`. Only continue at exit 0.

When every task in the chain has been implemented and the final build is green, go to **Step 5**.

---

## Step 3R — Routed mode (per-task model dispatch)

Used when Step 2.5 chose **routed mode**. You are already on the shared chain branch (Step 2). Implement the chain by dispatching one headless child per task, each on its own tier's model, **sequentially** (the next task builds on the previous one's commits, which are on disk once the child exits).

> **You do not write code here.** In routed mode the orchestrator's job is only: grep for the resume marker, resolve the model, dispatch, read the outcome, repeat. You never open files, edit, or `git commit` a task yourself. If you catch yourself implementing a task in-session, you've dropped out of routed mode — stop and dispatch it. The single exception is the explicit error **fallback** in 4·bullet 3 below.

For each task **T** in order:

1. **Already done?** `git log {branch} --grep "^Task #{T.task_number}:" -1` — if a commit exists, T is on the branch already → skip to the next task.
2. **Resolve T's model** from the `get_model_tiers` map using `T.difficulty` (default `standard`), and **T's tool scope** from `T.agent.allowed_tools`. Both are per task — tasks in one chain routinely resolve to different agents, so read them from T, never from the first task or from the chain.
3. **Dispatch one child** to implement only T on the current branch — run it from the repo dir so it inherits the `fabrio` connection:
   ```bash
   claude -p "/fabrio:feature-chain --step {T.task_number} --headless" --model "$T_MODEL" --permission-mode acceptEdits --allowedTools "{comma-joined T.agent.allowed_tools}"
   ```
   `--allowedTools` is fixed at spawn, so a child started without it silently runs on the machine's full allow-list — and this child is the one writing the code. Omit the flag only when `T.agent.allowed_tools` is empty (a workspace with no profiles yet), since an empty list would spawn a child that can do nothing.

   **Run it in the FOREGROUND and block until it exits — never background it.** In headless
   `-p` mode this orchestrator ends as soon as it stops emitting output, taking the `--step`
   child with it: no commit lands, the chain looks stalled, and the run exits 0 as though it
   had succeeded. There is no completion notification to wait for.

   This dispatch is **unconditionally headless** — nobody is watching that child regardless of whether the parent chain itself was invoked with `--headless`, so it always carries the flag. Wait for it to exit before the next task — never dispatch chain tasks in parallel (they share one checkout).
4. **After the child exits, read the outcome:**
   - A new `Task #{T.task_number}:` commit exists on `{branch}` **and** `get_task` shows T `in_progress` → success; continue to the next task.
   - **No** commit AND T has an open question / `is_blocked` / a posted decision → the child **held** on T. Go to **Step 4** (hold the chain; cascade to downstream tasks; no PR).
   - No commit and no block (the child errored) → **hold the chain and stop** — log `log_task_history { task_id: T.id, action: "dispatch_failed", notes: "Headless --step dispatch failed — chain held, no PR opened." }`, cascade `blocked_by_dependency` to the downstream tasks per Step 4, and open no PR. **Do not implement T inline.** The orchestrator runs on the cheapest model with the orchestrator's tool scope; implementing a task there silently discards its difficulty tier and its agent profile's `allowed_tools`, which a session cannot change mid-conversation. A held chain is re-runnable and costs one cycle; a chain half-built on the wrong model and the wrong tools is a PR nobody can trust.

When every task has its commit and the final build is green, go to **Step 5**. (Step 5's PR is always opened by the parent; per-task retrospectives are run by whoever implemented the task — Step 6.)

> **Unattended runs:** the `--step` children run headless (`-p`), so they need two things set up once (both covered in **Fabrio → Settings → API keys**): (1) an authenticated CLI that stays signed in — `claude auth login`, or a persistent `CLAUDE_CODE_OAUTH_TOKEN` / `ANTHROPIC_API_KEY` for scheduled runs; and (2) a **user-scope** allow-list in `~/.claude/settings.json` (`mcp__fabrio` plus `Bash(git:*)`, `Bash(gh:*)` (or `Bash(az:*)` on an Azure DevOps workspace), `Bash(npm run:*)`, `Bash(npx:*)`), since a child runs in each site's repo dir where project settings don't apply. The Step 2.5 probe catches both if they're missing. The inline fallback (Step 3R bullet 4's third case) runs in this same session, so it's HEADLESS only if this run itself was — an attended fallback may still chat-prompt as normal.

---

## `--step {n}` — single-task child (internal; used by Step 3R)

**Not a human entry point.** Routed mode dispatches this to implement exactly one task on the **already-checked-out** chain branch, then exit. It never sets up a branch, never resets to base, and never touches a PR. **HEADLESS is always true here** — this mode exists only to be run as a `claude -p` child (see the dispatch note in Step 3R), so 3e must never chat-prompt regardless of the `--headless` flag's literal presence.

1. Run Step 0's git-provider / MCP / source-root checks. **Skip** Step 1, Step 2, and Step 2.5 — the parent already owns grouping, the branch, and mode selection.
2. Confirm you're on a `feature/chain-*` branch: `git branch --show-current`. If not, stop with an error (`--step must be run by the chain orchestrator on an existing chain branch`) — do not create one.
3. Implement task `{n}` by running Step 3's **3b → 3k** for it (fetch/validate, open-question check, learnings/decisions, review-for-clarity, difficulty, plan checkpoint, claim, implement, commit with the `Task #{n}:` marker, build gate).
4. **On a block** (an open question, or you open a question/decision in 3c/3e, or an invalid status): do **not** commit; post the question as usual (which flags the task blocked) and **exit without error**. The parent detects the missing commit + block and holds the chain.
5. **On success**, after the build gate, run task `{n}`'s **retrospective** (Step 6 rubric) — you implemented it, so you have the context. Do **not** open a PR or set `under_review` (the parent does that once the whole chain lands). Then exit.

---

## Step 4 — Hold the Chain (a task needs input)

When a task **T** blocks in Step 3 (open question, freeform/decision question opened, missing/invalid prerequisite, or T already past implementation):

1. **Stop the chain now.** Do not attempt any task after T — they depend on it.
2. **Open no PR.** Leave the completed prefix committed on the shared branch; those tasks stay `in_progress`.
3. For **each task after T in the chain**, log the cascade:
   `log_task_history { task_id: {later.id}, action: "blocked_by_dependency", notes: "Held: chain depends on task #{T.task_number} which needs input. Both wait until #{T.task_number} is resolved." }`
4. Report and move to the next chain (multi-chain) / stop (single chain):
   ```
   ⏸  Chain feature/chain-{minN}-… held at #{T.task_number} — {reason}.
      {p} task(s) done on the branch (no PR yet); {q} downstream task(s) waiting.
      Resolve in the Questions/Decisions tab, then re-run the same invocation to resume.
   ```

The completed work is safe on the branch; Step 3a skips it on the next run.

---

## Step 5 — Open the Single PR (chain complete)

> **Checkpoint:** the PR url/number is saved to every task immediately — an interrupted run resumes via Step 2's chain-level check.

Push and open one PR covering the whole chain, targeting `$BASE_BRANCH`:
```bash
git push -u origin feature/chain-{minN}-{short-slug}
cat > /tmp/pr-body-chain-{minN}.md <<'PRBODY'
## Feature chain — {N} dependent tasks on one branch

**Site:** {site.name}

This PR implements the following tasks in order, each building on the previous:

### Task #{n1} — {title}
{summary} · Acceptance: {acceptance_criteria}

### Task #{n2} — {title}
{summary} · Acceptance: {acceptance_criteria}

_(…one section per task, in build order…)_

### Changes
{bullet list of the notable files created/modified across the chain}

### Testing
{how to verify the whole chain end-to-end}

---
🤖 Implemented by AI via Fabrio `/fabrio:feature-chain`
PRBODY
{PROVIDER.ops.create_pr}   # substitute {base_branch}, {branch}, {title}, {body_file}=/tmp/pr-body-chain-{minN}.md
```
**Capturing `pr_url`/`pr_number`:** GitHub returns no structured output from `create_pr`, so follow with `{PROVIDER.ops.view_pr}` scoped to the current branch (`gh pr view --json url,number`). Azure DevOps' `create_pr --output json` already returns the created PR's id/URL in the same call.

Then for **every** task in the chain: `update_task { task_id, fields: { pr_url, pr_number, status: "under_review" } }` (auto-logs `pr_linked` + the status change). Optional: `log_task_history { task_id, action: "ready_for_review", notes: "Chain PR #{pr_number} ready — {position} of {N} in chain (min #{minN})." }`.

---

## Step 6 — Retrospective (per task)

Same rubric as `/fabrio:feature-request` Step 11.5. The **implementer** records each task's retrospective, because it has the richest context:
- **Inline mode:** the parent runs the retrospective here, once per task in the chain.
- **Routed mode:** each `--step` child already ran its own task's retrospective right after committing (it loaded that task's learnings and did the work), so the parent does **not** repeat them — skip to Step 7.

For each task, record 0–3 generalizable learnings — `code_pattern` (codebase surprises), `pitfall` (first-attempt failures), `review_feedback` (for `changes_needed` tasks), `preference`, `process`. **Dedup** against that task's `loaded_learnings`: `reinforce_learning { learning_id }` if it restates one, else `record_learning { department: T.department, category, title, content, site_id: T.site_id (or omit for portfolio-wide), source_task_id: T.id }`. Always `log_task_history { task_id: T.id, action: "retrospective_saved", notes: "Recorded {N}, reinforced {M}" }`.

Regardless of mode, the parent may add one `process` learning when the chain itself taught it something (e.g. two tasks that should have been one, or an ordering that had to change).

---

## Step 7 — Output Summary

Per chain:
```
✅ Chain complete — {N} tasks on one branch.
  Site:   {site.name}   Branch: feature/chain-{minN}-{slug}
  Tasks:  #{n1} → #{n2} → #{n3}   (all under_review)
  PR:     {pr_url}
Awaiting human review — once approved, /fabrio:merge-task <any task #> merges the PR and marks the whole chain done.
```
Held chain:
```
⏸  Chain feature/chain-{minN}-… held at #{k} — {reason}. {p} done on branch, no PR. Resolve, then re-run.
```
After all chains (auto mode), print a one-line roll-up: `{C} chains — {done} opened PRs, {held} held, {skipped} already complete.`

---

## Error Recovery

On unexpected failure mid-chain: `log_task_history { action: "error", notes: "Chain (min #{minN}) failed at task #{T} / step {N}: {msg}" }`; leave git in a clean state (commit finished work with its `Task #{n}:` marker, or stash). **Do not** open a partial PR. Report the chain, task, and step. **Multi-chain:** log and continue to the next chain. **Resume** by re-running the same invocation — Step 2 finds the branch and Step 3a skips tasks already committed.
