---
description: "Merges an approved PR into the repo's base branch and marks the task as done. Deployment is triggered automatically by the merge."
---

# Merge Approved Task

Merge an approved PR into the repo's base branch and mark the task as done.

**Invocation:** `/fabrio:merge-task <task_number>`

All Fabrio data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials. The server scopes to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**.

---

## Prerequisites

**GitHub CLI** authenticated (`gh auth status`). If not, stop: `Error: GitHub CLI is not authenticated. Run: gh auth login`.

If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected:

> Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
> ```
> claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
> ```
> Restart Claude Code and re-invoke.

---

## Step 1 — Validate Invocation

Extract the task number. If none: `Error: Task number required. Usage: /fabrio:merge-task <task_number>`.

---

## Step 2 — Fetch Task

Call `get_task { task_number, include_history: true }`. If null: `Error: Task #{task_number} not found.` Store as `task`. Full site path = `{source_root}/{task.site.relative_path}` where `source_root` comes from the `FABRIO_SOURCE_ROOT` env var.

**If `FABRIO_SOURCE_ROOT` is unset**, tell the user it needs to be set (Claude Code reads it from its own environment, not Fabrio's `.env.local`) and give the fix, then ask for the path to use for this run:

> `FABRIO_SOURCE_ROOT` isn't set — set it once so I can find your repos:
> - **Recommended** — add to `~/.claude/settings.json` and restart Claude Code: `{ "env": { "FABRIO_SOURCE_ROOT": "C:\\Users\\you\\Source" } }` (macOS/Linux: `"/Users/you/Source"`).
> - **Or** set a real OS/shell env var before launching `claude`.
>
> For now, what's the absolute path to the folder that holds your site repos?

---

## Step 3 — Validate Status

Required status: **`approved`**. If anything else, log and stop:
`log_task_history { task_id: task.id, action: "skill_blocked", notes: "Merge skill rejected: status was \"{task.status}\", required \"approved\"" }`
Output: `Error: Task #{task_number} cannot be merged. Status is "{task.status}" — this skill only runs on "approved" tasks.` Stop.

---

## Step 4 — Validate PR Exists

Check `task.pr_number` and `task.pr_url` are set. If either is missing:
`log_task_history { task_id: task.id, action: "skill_blocked", notes: "Merge skill rejected: no PR linked to this task" }`
Output: `Error: Task #{task_number} has no linked PR.` Stop.

---

## Step 5 — Check PR State on GitHub

```bash
gh pr view {task.pr_number} --repo {owner/repo} --json state,mergeable,headRefName,url
```
Derive the repo from `task.pr_url` (e.g. `https://github.com/brandonturpin/FitPlan/pull/1` → `brandonturpin/FitPlan`).

- `state == MERGED` → `Error: PR #{n} is already merged.` — log and stop.
- `state == CLOSED` → `Error: PR #{n} is closed without merging.` — log and stop.
- `mergeable == CONFLICTING` → `Error: PR #{n} has merge conflicts … Resolve conflicts, then re-run.` — log and stop.

---

## Step 6 — Merge the PR

```bash
gh pr merge {task.pr_number} --repo {owner/repo} --squash --delete-branch
```
If the merge fails, log and stop: `log_task_history { task_id: task.id, action: "error", notes: "Merge failed: {error message}" }`.

---

## Step 7 — Pull the Base Branch Locally

Never assume `main` — resolve the base branch from the repo:
```bash
cd {site_path}
BASE_BRANCH=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
git checkout "$BASE_BRANCH"
git pull origin "$BASE_BRANCH"
```
If a local feature branch for this task remains after the remote branch was deleted, clean it up: `git branch -D {branch_name}`.

---

## Step 8 — Mark Task as Done

`update_task { task_id: task.id, fields: { status: "done", merged_at: "{current UTC ISO timestamp}" } }` (the status change is auto-logged). Add the merge note:
`log_task_history { task_id: task.id, action: "merged", notes: "PR #{task.pr_number} merged into {BASE_BRANCH} — deployment triggered automatically" }`

---

## Step 8.5 — Post-Merge Retrospective

This skill didn't write the code, so its retrospective is about the **review cycle** — visible only at merge time.

1. **Count change-request rounds** — from the `history` returned by `get_task` (Step 2), count entries with `action: "status_changed"` and `new_value: "changes_needed"`. That's the number of review cycles.

2. **If ≥ 1 cycle**: fetch the human PR comments and distill the recurring theme into ONE `review_feedback` learning, phrased as a rule (e.g. "Reviewer consistently asks for loading states on async buttons for {site}"):
   ```bash
   gh api repos/{owner}/{repo}/issues/{task.pr_number}/comments \
     --jq '[.[] | select(.user.type != "Bot")] | sort_by(.created_at) | .[] | .body'
   ```
   **Dedup first** — `list_learnings { department: task.department, site_id: task.site_id, include_portfolio: true, statuses: ["active"] }`. If the new learning restates an existing one, `reinforce_learning { learning_id }`. Otherwise `record_learning { site_id: task.site_id, department: task.department, source_task_id: task.id, category: "review_feedback", title: "{rule ≤200 chars}", content: "{directive ≤2000 chars}" }`.

3. **If 0 cycles**: record nothing — clean merges are the baseline.

4. **Log either way**: `log_task_history { task_id: task.id, action: "retrospective_saved", notes: "Merge retrospective: {N} review cycle(s), {0 or 1} learning recorded" }`.

---

## Step 9 — Output Summary

```
✅ Task #{task_number} merged and done.
  Title:   {task.title}
  Site:    {task.site.name}
  PR:      {task.pr_url}
  Branch:  deleted after merge
  Status:  done
The PR has been squash-merged into the repo's base branch. Deployment should be triggered automatically.
```

---

## Error Recovery

If any step fails unexpectedly:
1. Log it: `log_task_history { task_id: task.id, action: "error", notes: "Merge skill failed at step {N}: {error message}" }`.
2. Do NOT modify the task status — leave it `approved` so the skill can be re-run.
3. Report the error with the step number and what was attempted.
