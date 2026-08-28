---
name: feature-chain
description: "Implements a set of dependent feature tasks on one shared branch — each builds on the last — and opens a single PR for the whole chain. Auto-groups available tasks into chains, or takes an explicit ordered list."
---

# Feature Chain

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:ops-heartbeat`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

Implement **dependent** `execution_mode='repo'` tasks together: run them one-at-a-time on a **single shared branch** (each task builds on the previous task's commits), then open **one PR** covering the whole chain. This is for work that was split into several sequential tasks where you don't want to merge task 1 into the base branch just to start task 2 — build and test them together, check them in once.

- `$fabrio:feature-chain 12 13 14 15` — **explicit chain.** The order you give **is** the build order. All must be on the same site.
- `$fabrio:feature-chain --site <site_id|name>` — **auto-group** the available tasks for one site into dependency chains and run each.
- `$fabrio:feature-chain` — **auto-group across all sites** (one or more chains per site).

**Contrast with `$fabrio:feature-request`:** that skill gives every task its own branch and its own PR. Use it for independent tasks. Use **this** skill when tasks build on each other. A chain of one task produces exactly the same result as `feature-request` (one branch, one PR) — so auto-group mode safely handles standalone tasks too.

**Hold on block:** if any task in a chain needs human input (open question / a decision), the chain **stops there** and opens **no PR** — the completed work stays on the shared branch, and re-running resumes once the block is resolved.

**Resume:** re-running detects work already committed on the shared branch (per-task `Task #{n}:` commit markers) and picks up from the first task that isn't done yet.

**Model routing (automatic):** a session can't switch models mid-run, so whenever delegated dispatch is available, **every** task is implemented in its own delegated Codex agent on its tier's mapped model (a `light` task on the cheap model, a `heavy` one on the capable model), all on the same shared branch — regardless of what model this session is running. Only when delegated dispatch isn't available does the chain fall back to running inline on the current session's model (with a warning that tasks aren't routed). See Step 2.5.

---

## Step 0 — Setup

All Fabrio data access goes through the **`fabrio` MCP server** (tools named `mcp__fabrio__*`). There are **no** Supabase credentials or curl helpers — the server authenticates with a per-account API key and returns only the active account's data.

Run first; if any check fails, stop:

**Workspace git provider — do this before anything else, no default, ever (031).** Call `get_account_context`. **If `git_provider` is null, stop the entire run** (a chain can't start without a git host to open its PR against). Do not group tasks, create a branch, or edit a file first. Print exactly:
> `Error: No git provider is selected for this workspace. Set it in Fabrio → Settings → AI instructions, then re-run $fabrio:feature-chain.`

If `git_provider` is set, run its `ops.auth_check`. On failure, stop and print `git_provider.auth_hint` verbatim. **Never fall back to another provider, and never guess one from the git remote.**

Store the resolved provider as `PROVIDER` — every `PROVIDER.ops.*` reference in the rest of this skill means "run that command, substituting placeholders." `{repo}`/`{org}`/`{project}` come from `PROVIDER.coordinates` applied to the current repo's git remote; for GitHub these are inferred automatically by `gh` from the working directory. `get_task` (Step 3b) returns the same resolved provider as `task.account.git_provider`.

- If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected:
  > Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
  > ```
  > export FABRIO_API_KEY="fab_live_YOUR_KEY"\ncodex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY
  > ```
  > Restart Codex, open a new task, and re-invoke.

**Source root** — resolve it in this order: `FABRIO_SOURCE_ROOT`; then `~/.fabrio/config.json` → `source_root`; otherwise ask once and persist it using $fabrio:configure's merge-safe format. Site repos live at `{source_root}/{site.relative_path}`.

`BASE_BRANCH` is resolved **per repo** in Step 3 (never assume `main`).

Task history: use `log_task_history` for semantic milestones. Routine field edits (status, task_plan, pr_url, …) are logged automatically by `update_task`, so don't double-log those.

---

## Step 1 — Determine Chains

Produce an ordered list of **chains**, where each chain is an ordered list of tasks that all belong to **one site** (one branch + one PR = one repo).

### Explicit mode (task numbers given)
The arguments are one chain, **in the order given** = the build order. `get_task` each one and confirm they share a `site_id`. If any belong to a different site, **stop**: `Error: chain tasks span multiple sites (#{a} → {siteA}, #{b} → {siteB}). A chain must be one repo. Run them as separate chains.` Do not reorder an explicit chain — the user's order is authoritative.

### Auto-group mode (`--site` or bare)
Fetch workable tasks with `list_tasks { execution_mode: "repo", statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }` (add `site_id` when `--site` is given; resolve a site **name** to its id via `list_sites` first). **Only `repo` tasks can be chained** — a chain *is* a shared git branch, so a deliverable or an external action has nothing to build on. Department doesn't matter: a marketing landing page and a content blog post chain like any other repo work. Non-repo and unclassified tasks belong to `$fabrio:execute-task`. If none, output "No tasks are currently available to chain." and stop.

### `--delegated`
Accept this flag anywhere in the arguments (e.g. `$fabrio:feature-chain --site 3 --delegated`). `$fabrio:ops-heartbeat` always passes it on its dispatch (see below); a human typing the command directly normally doesn't. Set **HEADLESS = true** for this run if the flag is present — Step 3e reads this to decide how to raise a clarification question. (A `--step {n}` child sets its own HEADLESS unconditionally — see that section — since it skips this Step entirely.)

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

## Step 2.5 — Choose Execution Mode (MANDATORY)

Do this before implementing any task. Ensure every task has a persisted `difficulty` (`light`, `standard`, or `heavy`) and load `get_model_tiers` for the account's routing intent.

- **Routed mode (preferred):** when Codex agent delegation is available, delegate every task sequentially in Step 3R. Use the task's difficulty to select the closest available Codex model/reasoning tier; if an account mapping names a provider-specific model unavailable in Codex, preserve its quality intent (light/standard/heavy) and report the Codex model actually used.
- **Inline fallback:** only when agent delegation is unavailable. Run Step 3 in the current task and print that model-tier isolation is unavailable.

Print the decision and per-task routing before touching code. In routed mode, the orchestrator never implements task code itself; it only delegates, waits, verifies the commit and Fabrio state, and falls back inline for an individual task whose delegate failed without blocking or committing.
---

## Step 3 — Implement Each Task in the Chain — Inline fallback (in order)

**Enter this step only if Step 2.5 selected the inline fallback** (delegated dispatch unavailable). If Step 2.5 selected routed mode, go to **Step 3R** and do not implement here. For each task **T** in the chain, in order. This mirrors `$fabrio:feature-request` Steps 2–9 **minus** its per-task branch creation and per-task PR — you are already on the shared branch and stay on it.

### 3a — Already done on this branch? (resume marker)
```bash
git log {branch} --grep "^Task #{T.task_number}:" -1
```
If a commit exists, T is already implemented on the chain → output `↩  #{T.task_number} already on the branch — skipping to next.` and continue to the next task. (`claim_task` returning `{ claimed:false, current_status:"in_progress" }` for **your own** interrupted run is expected and not a conflict.)

### 3b — Fetch + validate
`get_task { task_number: T.task_number }` → task + `account` (workspace instructions + resolved git provider) + `site` + `questions` + `attachments`. Null → in a chain this breaks the build order; **hold the chain** (see Step 4) treating T as the blocker. Validate: `execution_mode == 'repo'` (else hold — a chain can't skip a prerequisite it has no way to build, and an unclassified task must go through `$fabrio:execute-task` first) and `status ∈ { ready, changes_needed, in_progress }` (`in_progress` only to resume). Note `task.department` (scopes learnings), `account.ai_context` (workspace rules — the chain's single shared branch must follow the workspace's naming convention if it sets one), `site.ai_context`, and `task.agent` (034 — its `instructions` are binding and its `skills` are the craft references for this task). Tasks in one chain may resolve to **different agents**; apply each task's own, not the first one's.

### 3c — Open questions → HOLD
If any `T.questions` has `status='open'`, the chain **holds at T** — go to Step 4. Everything after T depends on it, so don't attempt later tasks.

### 3d — Load learnings & decisions
`list_learnings { department: T.department, site_id: T.site_id, include_portfolio: true, statuses: ["active"], limit: 12 }` → `loaded_learnings` (treat as instructions: apply `code_pattern`/`preference`; check work against `pitfall`/`review_feedback`; follow `process`). `list_decisions { site_id: T.site_id, status: "decided" }` → `loaded_decisions` (binding — apply, don't re-ask).

### 3e — Review for clarity
Read the context layers widest-first — `account.ai_context` (workspace rules), then the department `playbook`, then `task.agent.instructions` (how this kind of work is done well), then `site.ai_context` (this repo), then T's `title`, `description`, `feature_summary`, `acceptance_criteria`, question threads, and any image `attachments` (view each `public_url`). All binding; the narrower layer wins a direct conflict. **No layer raises the autonomy ceiling.** For `changes_needed`, read the PR review comments. Ask: **can I implement this completely and correctly without a decision a human should make?** If `loaded_decisions` already covers the ambiguity, apply it and continue. **If clarification is still needed,** open it and **HOLD the chain** (Step 4):
- **(a) Structured decision** (choice between concrete options — prefer this): `create_decision { site_id: T.site_id, source_task_id: T.id, key, title, description, options:[…] }`, then `create_task_question { task_id: T.id, content, decision_id }`.
- **(b) Freeform question:** `create_task_question { task_id: T.id, content }` (auto-flags T blocked).

> **If HEADLESS** (the top-level `--delegated` flag, or this is a `--step` child — always delegated, see below): (a)/(b) are the *only* way to raise this — there is no one to answer a chat prompt, so record it and hold as just described. **If not HEADLESS** (a human is running this chain directly, no flag): asking in chat instead is fine — that's today's behavior and it's unchanged; posting via `create_task_question` is equally fine when you'd rather leave a record.

### 3f — Classify difficulty (if unset)
If `T.difficulty` is null, assign `light` / `standard` (default) / `heavy` and persist: `update_task { task_id: T.id, fields: { difficulty } }`.

### 3g — Plan (checkpoint)
Read the codebase, then save a per-task plan so an interrupted run has context: `update_task { task_id: T.id, fields: { task_plan: "<plan markdown>" } }` (auto-logs `plan_saved`). Cover Summary, Approach, Files, Database Changes (or "None"), Sub-Skills (including every skill in `task.agent.skills`), Learnings Applied, Agent Applied, Testing. Print it before writing code.

### 3h — Claim (concurrency guard)
`claim_task { task_id: T.id }` — atomically `ready|changes_needed → in_progress`.
- `{ claimed: true }` → continue.
- `{ claimed: false, current_status }` → `in_progress` is your own resume (proceed); `under_review`/`approved`/`done` means it was handled elsewhere — that breaks the chain's assumptions, so **hold** and tell the user T is already past implementation.

### 3i — Implement (on the shared branch)
Follow the plan; read adjacent files and match existing patterns exactly. DB changes → **new numbered migration** in `supabase/migrations/` + update `supabase/schema.sql`. Type-check with `npx tsc --noEmit` as you go. **Sub-skills — invoke when applicable:** `$frontend-design`, `$react-best-practices`, `$web-design-guidelines`, `$composition-patterns`, `$ux-review`.

**Fabrio conventions:** dark mode only (`zinc-950` → `zinc-900` → `zinc-800`); success `emerald-400`, destructive `rose-400`; API routes follow `app/api/sites/` & `app/api/tasks/`; hooks follow the SWR pattern in `hooks/useSites.ts`.

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

## Step 3R — Routed mode (per-task Codex agent delegation)

You are already on the shared chain branch. Delegate tasks **sequentially** because they share one checkout and each task builds on the prior task's commit.

For each task **T** in order:

1. Check `git log {branch} --grep "^Task #{T.task_number}:" -1`; skip an existing marker.
2. Resolve the closest available Codex model/reasoning tier from T's difficulty and the account's tier intent.
3. Delegate `$fabrio:feature-chain --step {T.task_number}` to one Codex agent in the repo working directory. Give it the selected tier, current branch, task number, and the instruction not to open a PR. Wait for completion before continuing.
4. Verify the result:
   - New task commit plus Fabrio status `in_progress` → continue.
   - Open question, `is_blocked`, or posted decision → hold the chain at Step 4.
   - No commit and no durable block → run Step 3's 3b–3k inline for T and log `dispatch_fallback`.

When every task has its commit and the final build is green, go to Step 5. The parent always opens the single PR.
---

## `--step {n}` — single-task child (internal; used by Step 3R)

**Not a human entry point.** Routed mode dispatches this to implement exactly one task on the **already-checked-out** chain branch, then exit. It never sets up a branch, never resets to base, and never touches a PR. **HEADLESS is always true here** — this mode exists only to be run as a `delegate the referenced $fabrio:* skill to a Codex sub-agent` child (see the dispatch note in Step 3R), so 3e must never chat-prompt regardless of the `--delegated` flag's literal presence.

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
🤖 Implemented by AI via Fabrio `$fabrio:feature-chain`
PRBODY
{PROVIDER.ops.create_pr}   # substitute {base_branch}, {branch}, {title}, {body_file}=/tmp/pr-body-chain-{minN}.md
```
**Capturing `pr_url`/`pr_number`:** GitHub returns no structured output from `create_pr`, so follow with `{PROVIDER.ops.view_pr}` scoped to the current branch (`gh pr view --json url,number`). Azure DevOps' `create_pr --output json` already returns the created PR's id/URL in the same call.

Then for **every** task in the chain: `update_task { task_id, fields: { pr_url, pr_number, status: "under_review" } }` (auto-logs `pr_linked` + the status change). Optional: `log_task_history { task_id, action: "ready_for_review", notes: "Chain PR #{pr_number} ready — {position} of {N} in chain (min #{minN})." }`.

---

## Step 6 — Retrospective (per task)

Same rubric as `$fabrio:feature-request` Step 11.5. The **implementer** records each task's retrospective, because it has the richest context:
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
Awaiting human review — once approved, $fabrio:merge-task <any task #> merges the PR and marks the whole chain done.
```
Held chain:
```
⏸  Chain feature/chain-{minN}-… held at #{k} — {reason}. {p} done on branch, no PR. Resolve, then re-run.
```
After all chains (auto mode), print a one-line roll-up: `{C} chains — {done} opened PRs, {held} held, {skipped} already complete.`

---

## Error Recovery

On unexpected failure mid-chain: `log_task_history { action: "error", notes: "Chain (min #{minN}) failed at task #{T} / step {N}: {msg}" }`; leave git in a clean state (commit finished work with its `Task #{n}:` marker, or stash). **Do not** open a partial PR. Report the chain, task, and step. **Multi-chain:** log and continue to the next chain. **Resume** by re-running the same invocation — Step 2 finds the branch and Step 3a skips tasks already committed.
