---
description: "Implements a set of dependent feature tasks on one shared branch — each builds on the last — and opens a single PR for the whole chain. Auto-groups available tasks into chains, or takes an explicit ordered list."
---

# Feature Chain

Implement **dependent** `type='feature_request'` tasks together: run them one-at-a-time on a **single shared branch** (each task builds on the previous task's commits), then open **one PR** covering the whole chain. This is for work that was split into several sequential tasks where you don't want to merge task 1 into the base branch just to start task 2 — build and test them together, check them in once.

- `/fabrio:feature-chain 12 13 14 15` — **explicit chain.** The order you give **is** the build order. All must be on the same site.
- `/fabrio:feature-chain --site <site_id|name>` — **auto-group** the available tasks for one site into dependency chains and run each.
- `/fabrio:feature-chain` — **auto-group across all sites** (one or more chains per site).

**Contrast with `/fabrio:feature-request`:** that skill gives every task its own branch and its own PR. Use it for independent tasks. Use **this** skill when tasks build on each other. A chain of one task produces exactly the same result as `feature-request` (one branch, one PR) — so auto-group mode safely handles standalone tasks too.

**Hold on block:** if any task in a chain needs human input (open question / a decision), the chain **stops there** and opens **no PR** — the completed work stays on the shared branch, and re-running resumes once the block is resolved.

**Resume:** re-running detects work already committed on the shared branch (per-task `Task #{n}:` commit markers) and picks up from the first task that isn't done yet.

---

## Step 0 — Setup

All Fabrio data access goes through the **`fabrio` MCP server** (tools named `mcp__fabrio__*`). There are **no** Supabase credentials or curl helpers — the server authenticates with a per-account API key and returns only the active account's data.

Run first; if any check fails, stop:

```bash
gh auth status                # GitHub CLI must be authed
```
- If `gh` isn't authed, stop and tell the user to `gh auth login`.
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
Fetch workable tasks with `list_tasks { type: "feature_request", statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }` (add `site_id` when `--site` is given; resolve a site **name** to its id via `list_sites` first). Only `feature_request` is implemented — `marketing`/content tasks are tracked but not auto-implemented. If none, output "No tasks are currently available to chain." and stop.

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
BASE_BRANCH=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
```

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
gh api repos/{owner}/{repo}/issues/{pr_number}/comments --jq '[.[] | select(.user.type != "Bot")] | sort_by(.created_at)'
git log origin/{branch} -1 --format="%aI"
```
If there's newer feedback, read it (it applies to whichever task(s) it names), re-implement on the branch, run Step 8's build gate, push, and stop. If not, output `⏭  Chain feature/chain-{minN}-… already complete (PR #{pr_number}). Skipping.` and move to the next chain.

---

## Step 3 — Implement Each Task in the Chain (in order)

For each task **T** in the chain, in order. This mirrors `/fabrio:feature-request` Steps 2–9 **minus** its per-task branch creation and per-task PR — you are already on the shared branch and stay on it.

### 3a — Already done on this branch? (resume marker)
```bash
git log {branch} --grep "^Task #{T.task_number}:" -1
```
If a commit exists, T is already implemented on the chain → output `↩  #{T.task_number} already on the branch — skipping to next.` and continue to the next task. (`claim_task` returning `{ claimed:false, current_status:"in_progress" }` for **your own** interrupted run is expected and not a conflict.)

### 3b — Fetch + validate
`get_task { task_number: T.task_number }` → task + `site` + `questions` + `attachments`. Null → in a chain this breaks the build order; **hold the chain** (see Step 4) treating T as the blocker. Validate: `type == 'feature_request'` (else hold — a chain can't skip a non-code prerequisite) and `status ∈ { ready, changes_needed, in_progress }` (`in_progress` only to resume). Note `task.department` (scopes learnings) and `site.ai_context`.

### 3c — Open questions → HOLD
If any `T.questions` has `status='open'`, the chain **holds at T** — go to Step 4. Everything after T depends on it, so don't attempt later tasks.

### 3d — Load learnings & decisions
`list_learnings { department: T.department, site_id: T.site_id, include_portfolio: true, statuses: ["active"], limit: 12 }` → `loaded_learnings` (treat as instructions: apply `code_pattern`/`preference`; check work against `pitfall`/`review_feedback`; follow `process`). `list_decisions { site_id: T.site_id, status: "decided" }` → `loaded_decisions` (binding — apply, don't re-ask).

### 3e — Review for clarity
Read `site.ai_context` first (foundational), then T's `title`, `description`, `feature_summary`, `acceptance_criteria`, question threads, and any image `attachments` (view each `public_url`). For `changes_needed`, read the PR review comments. Ask: **can I implement this completely and correctly without a decision a human should make?** If `loaded_decisions` already covers the ambiguity, apply it and continue. **If clarification is still needed,** open it and **HOLD the chain** (Step 4):
- **(a) Structured decision** (choice between concrete options — prefer this): `create_decision { site_id: T.site_id, source_task_id: T.id, key, title, description, options:[…] }`, then `create_task_question { task_id: T.id, content, decision_id }`.
- **(b) Freeform question:** `create_task_question { task_id: T.id, content }` (auto-flags T blocked).

### 3f — Classify difficulty (if unset)
If `T.difficulty` is null, assign `light` / `standard` (default) / `heavy` and persist: `update_task { task_id: T.id, fields: { difficulty } }`.

### 3g — Plan (checkpoint)
Read the codebase, then save a per-task plan so an interrupted run has context: `update_task { task_id: T.id, fields: { task_plan: "<plan markdown>" } }` (auto-logs `plan_saved`). Cover Summary, Approach, Files, Database Changes (or "None"), Sub-Skills, Learnings Applied, Testing. Print it before writing code.

### 3h — Claim (concurrency guard)
`claim_task { task_id: T.id }` — atomically `ready|changes_needed → in_progress`.
- `{ claimed: true }` → continue.
- `{ claimed: false, current_status }` → `in_progress` is your own resume (proceed); `under_review`/`approved`/`done` means it was handled elsewhere — that breaks the chain's assumptions, so **hold** and tell the user T is already past implementation.

### 3i — Implement (on the shared branch)
Follow the plan; read adjacent files and match existing patterns exactly. DB changes → **new numbered migration** in `supabase/migrations/` + update `supabase/schema.sql`. Type-check with `npx tsc --noEmit` as you go. **Sub-skills — invoke when applicable:** `/frontend-design`, `/react-best-practices`, `/web-design-guidelines`, `/composition-patterns`, `/ux-review`.

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
gh pr create --base "$BASE_BRANCH" --title "Chain #{minN}: {chain theme} ({N} tasks)" --body "$(cat <<'PRBODY'
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
)"
gh pr view --json url,number          # capture pr_url, pr_number
```

Then for **every** task in the chain: `update_task { task_id, fields: { pr_url, pr_number, status: "under_review" } }` (auto-logs `pr_linked` + the status change). Optional: `log_task_history { task_id, action: "ready_for_review", notes: "Chain PR #{pr_number} ready — {position} of {N} in chain (min #{minN})." }`.

---

## Step 6 — Retrospective (per task)

Run once **per task in the chain** (same rubric as `/fabrio:feature-request` Step 11.5). For each T, record 0–3 generalizable learnings — `code_pattern` (codebase surprises), `pitfall` (first-attempt failures), `review_feedback` (for `changes_needed` tasks), `preference`, `process`. **Dedup** against that task's `loaded_learnings`: `reinforce_learning { learning_id }` if it restates one, else `record_learning { department: T.department, category, title, content, site_id: T.site_id (or omit for portfolio-wide), source_task_id: T.id }`. Always `log_task_history { task_id: T.id, action: "retrospective_saved", notes: "Recorded {N}, reinforced {M}" }`.

Also worth a `process` learning when the chain itself taught you something (e.g. two tasks that should have been one, or an ordering that had to change).

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
