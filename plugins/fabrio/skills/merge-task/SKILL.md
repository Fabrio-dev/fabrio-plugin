---
name: merge-task
description: "Closes out an approved task — squash-merges its PR when it has one, or simply records completion for tasks whose deliverable was a document or an external action."
---

# Complete Approved Task

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:ops-heartbeat`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

Close out an approved task. **The single completion path for every execution mode** — what it does depends on whether the task has a PR:

| task has | this skill |
|---|---|
| `pr_number` (repo work) | validates + squash-merges the PR, pulls base, marks done |
| no PR (`artifact` / `external`) | records completion (plus any external result the human supplies) and marks done |

**Invocation:** `$fabrio:merge-task <task_number>`

All Fabrio data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials. The server scopes to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**.

---

## Prerequisites

**Workspace git provider** (031) — **only required for a task that has a PR.** Checked at Step 5, not up front: an `artifact`/`external` task never touches a git host and must not be blocked on config it doesn't use. When it is needed: if the workspace has no `git_provider` selected, or its CLI isn't authenticated, Step 5 stops with the exact remediation.

If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> export FABRIO_API_KEY="fab_live_YOUR_KEY"\ncodex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY
> ```
> Restart Codex, open a new task, and re-invoke.

---

## Step 1 — Validate Invocation

Extract the task number. If none: `Error: Task number required. Usage: $fabrio:merge-task <task_number>`.

---

## Step 2 — Fetch Task

Call `get_task { task_number, include_history: true }`. If null: `Error: Task #{task_number} not found.` Store as `task`. Note `task.execution_mode`, `task.pr_number`, `task.account.git_provider` (the workspace's resolved git provider — read at Step 5, only for tasks with a PR) and (for non-repo tasks) `task.deliverable`.

**Resolve `source_root` only if `task.pr_number` is set** — a task with no PR has no local repo to touch, so skip this entirely rather than prompting for a path the run will never use. Full site path = `{source_root}/{task.site.relative_path}`. Resolve in this order (stop at the first that yields a value):
1. The `FABRIO_SOURCE_ROOT` env var, if set (back-compat — power users may keep this).
2. Else read `~/.fabrio/config.json` (`%USERPROFILE%\.fabrio\config.json` on Windows, `$HOME/.fabrio/config.json` otherwise) and use its `source_root`.
3. Else **ask once and persist it**: prompt *"What's the absolute path to the folder that holds your site repos?"*, then write `source_root` into `~/.fabrio/config.json` (create the `.fabrio` dir + file, merging with any existing keys — see `$fabrio:configure` Step 2 for the exact write), use it for this run, and tell the user they can re-run `$fabrio:configure` to change it.

Skip the prompt entirely if step 1 or 2 supplied a path.

---

## Step 3 — Validate Status

Required status: **`approved`**. If anything else, log and stop:
`log_task_history { task_id: task.id, action: "skill_blocked", notes: "Merge skill rejected: status was \"{task.status}\", required \"approved\"" }`
Output: `Error: Task #{task_number} cannot be merged. Status is "{task.status}" — this skill only runs on "approved" tasks.` Stop.

> **Feature chains:** if this task shares its PR with others (a `$fabrio:feature-chain` run), approving and invoking merge on **any one** of them merges the shared PR and closes the whole chain (Step 8). The invoked task must itself be `approved`; its siblings may still be `under_review` — the merge is the human's go-ahead for the entire PR. Chains are repo-only, so this never applies to an `artifact`/`external` task.

---

## Step 4 — Route on the Completion Shape

**If `task.pr_number` AND `task.pr_url` are set** → repo work. Continue to Step 5.

**If neither is set** → the deliverable was a document or an external action. Verify there is actually something to complete: `task.deliverable` must be non-empty. If it's empty, the task was approved with no output at all, which is a mistake worth stopping on:
`log_task_history { task_id: task.id, action: "skill_blocked", notes: "Completion rejected: no PR and no deliverable — nothing was produced." }`
Output: `Error: Task #{task_number} has neither a PR nor a deliverable. Run $fabrio:execute-task {task_number} to produce one.` Stop.

Otherwise **skip Steps 5–7** (there is no PR to merge and no branch to clean) and go to **Step 7.5**.

**If exactly one of `pr_number`/`pr_url` is set** → the pair is inconsistent, which usually means a half-finished PR step:
`log_task_history { task_id: task.id, action: "skill_blocked", notes: "Completion rejected: pr_url/pr_number are inconsistent" }`
Output: `Error: Task #{task_number} has an incomplete PR link (pr_url={...}, pr_number={...}). Re-run $fabrio:feature-request {task_number} to repair it.` Stop.

---

## Step 5 — Check PR State on the Git Provider

*(repo tasks only)*

**Workspace git provider (031) — no default, ever.** `task.account.git_provider` (from Step 2). **If it is null, stop** — the workspace's provider setting was cleared since the PR was opened:
> `Error: No git provider is selected for this workspace. Set it in Fabrio → Settings → AI instructions, then re-run $fabrio:merge-task {task_number}.`

Otherwise run its `ops.auth_check`; on failure stop and print `git_provider.auth_hint` verbatim. Store the resolved provider as `PROVIDER` for the rest of this skill.

```bash
{PROVIDER.ops.view_pr}   # substitute {pr_number} and {repo}/{org}/{project}
```
Derive `{repo}`/`{org}`/`{project}` from `task.pr_url` per `PROVIDER.coordinates` (e.g. for GitHub, `https://github.com/brandonturpin/FitPlan/pull/1` → `repo = brandonturpin/FitPlan`).

**Reading the response:** GitHub uses `state`/`mergeable`/`headRefName`; Azure DevOps uses `status`/`mergeStatus`/`sourceRefName` (no direct "conflicting" boolean — a non-completed PR with conflicts shows `mergeStatus: "conflicts"`).

- Already merged (GitHub `state == MERGED`; Azure DevOps `status == completed`) → `Error: PR #{n} is already merged.` — log and stop.
- Closed without merging (GitHub `state == CLOSED`; Azure DevOps `status == abandoned`) → `Error: PR #{n} is closed without merging.` — log and stop.
- Has conflicts (GitHub `mergeable == CONFLICTING`; Azure DevOps `mergeStatus == conflicts`) → `Error: PR #{n} has merge conflicts … Resolve conflicts, then re-run.` — log and stop.

---

## Step 6 — Merge the PR

```bash
{PROVIDER.ops.merge_pr}   # substitute {pr_number} and {repo}/{org}/{project}
```
If the merge fails, log and stop: `log_task_history { task_id: task.id, action: "error", notes: "Merge failed: {error message}" }`.

---

## Step 7 — Pull the Base Branch Locally

Never assume `main` — resolve the base branch from the repo:
```bash
cd {site_path}
BASE_BRANCH=$({PROVIDER.ops.default_branch})   # Azure DevOps: strip the "refs/heads/" prefix
git checkout "$BASE_BRANCH"
git pull origin "$BASE_BRANCH"
```
If a local feature branch for this task remains after the remote branch was deleted, clean it up: `git branch -D {branch_name}`.

---

## Step 7.5 — Record the External Result

*(non-repo tasks only — skip for repo tasks, and skip entirely for `artifact` mode)*

An `external` task's deliverable was a package for a human to perform. If the invocation carried a result the human wants recorded — a post URL, a campaign id, a send timestamp — persist it so the next cadence has continuity:

`update_task { task_id: task.id, fields: { deliverable_meta: { …existing meta…, external_url: "{url}", published_at: "{UTC ISO}" } } }`

Merge into the existing `deliverable_meta` rather than replacing it, so the channel/scheduling context the executor wrote survives. **Never** put a credential or token in it.

If no result was supplied, don't ask for one and don't invent one — record completion without it and mention in the summary that no external reference was captured.

> **This skill never performs the external action either.** Reaching this step means the human already did it; you are only writing down that it happened.

---

## Step 8 — Mark Task(s) as Done

### Repo tasks — close the whole PR

A single PR can cover **several** tasks — a `$fabrio:feature-chain` run puts a whole dependency chain on one branch and links every task in it to the same `pr_number`. Merging that PR completes **all** of them, so mark every task on this PR done, not just the invoked one.

Find the siblings: `list_tasks { pr_number: task.pr_number }`. This returns every task sharing the merged PR (just the one for a normal `$fabrio:feature-request`; the whole chain for `$fabrio:feature-chain`). For **each** returned task whose status isn't already `done`:

`update_task { task_id: {sibling.id}, fields: { status: "done", merged_at: "{current UTC ISO timestamp}" } }` (the status change is auto-logged). The merged PR is the source of truth, so a sibling still in `under_review` does **not** need to be individually `approved` first. Add the merge note per task:
`log_task_history { task_id: {sibling.id}, action: "merged", notes: "PR #{task.pr_number} merged into {BASE_BRANCH} — deployment triggered automatically" }`

Track `{merged_count}` = how many tasks you marked done (1 for a normal task, N for a chain).

### Non-repo tasks — close just this one

There is no PR, so there is **no sibling lookup**: `list_tasks { pr_number: … }` with a null PR would match unrelated tasks. Close only the invoked task:

`update_task { task_id: task.id, fields: { status: "done", merged_at: "{current UTC ISO timestamp}" } }`

`merged_at` is the completion timestamp here — nothing was merged, but it's the field the dashboard and analytics already read as "when did this finish", so keep it consistent rather than leaving it null. Then:
`log_task_history { task_id: task.id, action: "completed", notes: "{artifact: 'Deliverable approved and task closed' | external: 'External action performed by human; task closed{ — result: {external_url}}'}" }`

Set `{merged_count}` = 1.

---

## Step 8.5 — Post-Merge Retrospective

This skill didn't produce the work, so its retrospective is about the **review cycle** — visible only at completion time. It runs for every mode; only where the feedback lives differs.

1. **Count change-request rounds** — from the `history` returned by `get_task` (Step 2), count entries with `action: "status_changed"` and `new_value: "changes_needed"`. That's the number of review cycles.

2. **If ≥ 1 cycle**: gather the human feedback, then distill the recurring theme into ONE `review_feedback` learning phrased as a rule (e.g. "Reviewer consistently asks for loading states on async buttons for {site}", or "Marketing copy for {site} must avoid superlatives").

   **Repo tasks** — read the PR comments:
   ```bash
   {PROVIDER.ops.pr_comments}   # substitute {pr_number} and {repo}/{org}/{project}; flatten Azure DevOps' threads to one chronological list
   ```
   **Non-repo tasks** — the feedback is in Fabrio, not GitHub: read the `history` entries with `action: "review_feedback"` (written by the reviewer from the Deliverable tab), plus any human (`role: "human"`) messages in `task.questions[].messages`.

   **Dedup first** — `list_learnings { department: task.department, site_id: task.site_id, include_portfolio: true, statuses: ["active"] }`. If the new learning restates an existing one, `reinforce_learning { learning_id }`. Otherwise `record_learning { site_id: task.site_id, department: task.department, source_task_id: task.id, category: "review_feedback", title: "{rule ≤200 chars}", content: "{directive ≤2000 chars}" }`.

3. **If 0 cycles**: record nothing — clean first passes are the baseline.

4. **Log either way**: `log_task_history { task_id: task.id, action: "retrospective_saved", notes: "Merge retrospective: {N} review cycle(s), {0 or 1} learning recorded" }`.

---

## Step 9 — Output Summary

**Single repo task** (`{merged_count}` == 1, had a PR):
```
✅ Task #{task_number} merged and done.
  Title:   {task.title}
  Site:    {task.site.name}
  PR:      {task.pr_url}
  Branch:  deleted after merge
  Status:  done
The PR has been squash-merged into the repo's base branch. Deployment should be triggered automatically.
```
**Artifact task** (no PR):
```
✅ Task #{task_number} complete.
  Title:   {task.title}
  Site:    {task.site.name}
  Output:  deliverable approved (on the task — no repo change)
  Status:  done
```
**External task** (no PR):
```
✅ Task #{task_number} complete.
  Title:   {task.title}
  Site:    {task.site.name}
  Channel: {deliverable_meta.channel or "—"}
  Result:  {external_url if recorded, else "no external reference captured"}
  Status:  done
Recorded as performed by you — this skill never publishes or sends anything itself.
```
**Chain** (`{merged_count}` > 1) — the merged PR closed the whole chain:
```
✅ Chain merged — {merged_count} tasks done via PR #{task.pr_number}.
  Tasks:   #{n1}, #{n2}, #{n3}   (all → done)
  Site:    {task.site.name}
  PR:      {task.pr_url}
  Branch:  deleted after merge
The PR has been squash-merged into the repo's base branch. Deployment should be triggered automatically.
```

---

## Error Recovery

If any step fails unexpectedly:
1. Log it: `log_task_history { task_id: task.id, action: "error", notes: "Completion skill failed at step {N}: {error message}" }`.
2. Do NOT modify the task status — leave it `approved` so the skill can be re-run.
3. Report the error with the step number and what was attempted.
