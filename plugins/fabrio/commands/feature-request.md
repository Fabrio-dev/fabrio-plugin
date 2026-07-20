---
description: "Implements feature request tasks end-to-end — single task or all available tasks, with full resume support."
---

# Feature Request

Implement `type='feature_request'` tasks from the Fabrio task system end-to-end.

- `/feature-request 42` — implement one task by number.
- `/feature-request` — implement ALL available tasks (batch).

**Resume:** re-running a task detects existing work (saved plan, existing PR) and picks up where it left off instead of starting over.

---

## Step 0 — Setup

All Fabrio data access goes through the **`fabrio` MCP server** (tools named `mcp__fabrio__*`). There are **no** Supabase credentials, curl helpers, or connection strings anymore — the server authenticates with a per-account API key and returns only the active account's data.

Run first; if any check fails, stop:

```bash
gh auth status                # GitHub CLI must be authed
```
- If `gh` isn't authed, stop and tell the user to `gh auth login`.
- If the `mcp__fabrio__*` tools aren't available, stop and tell the user to connect an account: `scripts/use-account.ps1 <name>` (Windows) or `scripts/use-account.sh <name>`. This is also how they switch which account you operate on.

**Source root** — the site repos live at `{source_root}/{site.relative_path}`. Read `source_root` from the `FABRIO_SOURCE_ROOT` environment variable; if it's unset, ask the user once and use that for the run. (`get_site` / `get_task` return `relative_path`.)

**Base branch** — never assume `main`; resolve at runtime from inside each target repo and use it for every checkout/pull/`--base`:
```bash
BASE_BRANCH=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
```

Task history: use `log_task_history` for semantic milestones. Routine field edits (status, task_plan, pr_url, …) are logged automatically by `update_task`, so don't double-log those.

---

## Step 1 — Determine Mode

**Single-task mode** (argument given): extract the number, go to Step 2 for that task.

**Batch mode** (no argument): fetch all workable code tasks with `list_tasks` — `{ type: "feature_request", statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }`. Only `feature_request` is implemented here; `marketing`/content tasks are tracked but not auto-implemented. If none, output "No tasks are currently available to work on." and stop. Otherwise list them, then run **Steps 2–12 for each in order**, returning here after each.

> **Model routing note:** batch mode runs every task in the current session's model — it does not switch per task. Tier-aware routing is `/ops-heartbeat`'s job (it dispatches each task as its own headless `/feature-request {n}` on the resolved model). Step 5.5 still classifies unset tiers so those runs route correctly.

Before starting each task, reset git to a clean base inside that task's repo:
```bash
git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
```

---

## Step 2 — Fetch Full Task Data

Call `get_task` with `{ task_number }`. It returns the task plus `site` (id, name, relative_path, live_url, ai_context), `questions` (threads with messages), and `attachments` in one call. If it returns null, output `Error: Task #{task_number} not found.` and stop/skip. Note `task.department` (scopes learnings) and `task.site.ai_context`. Full site path = `{source_root}/{task.site.relative_path}`.

---

## Step 3 — Validate Type & Status

**Type guard:** only `type='feature_request'` is implemented. If `task.type == 'marketing'`, log and skip (batch) / stop (single):
`log_task_history { task_id, action: "skill_skipped", notes: "Not implemented by feature-request: type is marketing (tracked, not auto-implemented)." }`
Output `⏭  Task #{task_number} is a marketing/content task — tracked, not auto-implemented.`

**Status:** allowed = `ready`, `changes_needed`, and `in_progress` (only to resume a run interrupted mid-implementation — see Step 7). Anything else → `log_task_history` with `skill_skipped` (batch, continue) or `skill_blocked` (single, stop), note `status was "{task.status}", required "ready" or "changes_needed"`; in single mode output the error and stop.

---

## Step 3.5 — Resume Detection

Check existing state before doing work, in order:

**Already complete** — `status == 'under_review'` AND `pr_url` set: output `⏭  Task #{n} already complete … Skipping.`; next (batch) / stop (single).

**PR exists, status not updated** — `pr_url` AND `pr_number` set AND `status != 'under_review'`: before skipping to Step 11, check for unread review feedback:
```bash
gh api repos/{owner}/{repo}/issues/{task.pr_number}/comments --jq '[.[] | select(.user.type != "Bot")] | sort_by(.created_at)'
gh pr view {task.pr_number} --repo {owner}/{repo} --json headRefOid,updatedAt
git log origin/{branch_name} -1 --format="%aI"
```
Derive `{owner}/{repo}` from `pr_url`. **If the latest human comment is newer than the latest branch commit**, treat it as `changes_needed`: read every comment (they are the required changes), output `↩  … has new review comments since last push. Implementing.`, check out the branch, and run Step 7 → 8 → 9 → 10 (push) → 11. **Otherwise** output `↩  … Resuming from Step 11.` and skip straight to Step 11.

**Plan exists, no PR** — `task_plan` set AND `pr_url` null: output `↩  … Resuming from Step 7.`, `log_task_history { action: "skill_resumed" }`, skip Steps 4–6, go to Step 7 using the existing plan.

**Fresh task** — no plan, no PR: proceed from Step 4.

---

## Step 4 — Check for Open Questions

If any `task.questions` has `status='open'`, log a block and stop/skip:
`log_task_history { task_id, action: "skill_blocked", notes: "{N} open question(s) must be answered before implementation" }`
Batch: `⏭  Task #{n} has {N} unanswered question(s) — skipping.` Single: list each open question's first AI message and stop with instructions to answer in the Questions tab.

---

## Step 4.5 — Load Learnings & Decisions

Call `list_learnings` with `{ department: task.department, site_id: task.site_id, include_portfolio: true, statuses: ["active"], limit: 12 }`. Store as `loaded_learnings` — treat as instructions, not suggestions: apply `code_pattern`/`preference` while reading and writing code; actively check work against `pitfall`/`review_feedback` (the reviewer WILL re-flag them); follow `process`. No rows is fine (first runs have no memory).

Then call `list_decisions` with `{ site_id: task.site_id, status: "decided" }`. Treat a `decided` decision as binding: apply its `chosen_option_key`/`chosen_rationale` instead of re-asking. A `chosen_option_key` of `__custom` means the human wrote their own answer — follow `chosen_rationale`. Store as `loaded_decisions`.

---

## Step 5 — Review for Clarity

Read `task.site.ai_context` **first if present** (foundational), then `task.title`, `description`, `feature_summary`, `acceptance_criteria`, and all question threads. If `task.attachments` is non-empty, view each image `public_url` (via WebFetch or an image tool) before implementing — treat it as a visual spec (note PDFs but focus on images).

For `changes_needed`, read ALL PR review comments chronologically:
```bash
gh api repos/{owner}/{repo}/issues/{task.pr_number}/comments --jq '[.[] | select(.user.type != "Bot")] | sort_by(.created_at) | .[] | "[\(.created_at)] \(.user.login): \(.body)"'
```
Comments after the last branch push are the most recent required changes.

Ask: **can I implement this completely and correctly without making assumptions a human should decide?** Watch for missing scope, edge-case behavior, data shapes / API contracts / schema changes, or approach-defining decisions.

First check `loaded_decisions`: if a `decided` decision already covers the ambiguity, **apply it and continue — do not re-ask.**

**If clarification is still needed,** pick the right shape:

**(a) Structured decision** — the ambiguity is a choice between concrete options. Prefer this: it renders as selectable option cards and persists as reusable memory. Call `create_decision` (idempotent — get-or-create on the site + key, so re-runs and sibling tasks reuse the same decision):
```
create_decision {
  site_id: task.site_id, source_task_id: task.id,
  key: "{stable-slug e.g. billing-activity-value}",
  title: "{the decision, one line}", description: "{context + why it matters}",
  options: [
    { key: "A", label: "{short label}", description: "{trade-offs}", recommended: true, example: "{short worked example}" },
    { key: "B", label: "{short label}", description: "{…}" }
  ]
}
```
Then attach a decision question to THIS task with `create_task_question { task_id: task.id, content: "{one-line framing}", decision_id: <the decision id> }`. When the human picks an option (in the UI), the decision is recorded and **every** task blocked on that `decision_id` is unblocked at once.

**(b) Freeform question** — open-ended. Call `create_task_question { task_id: task.id, content: "{your question}" }` (opening a question auto-flags the task blocked). For a direct follow-up on an existing thread, use `post_question_message { question_id, content, reopen: true }` instead.

Then skip (batch) / stop (single). Batch: `⏭  Task #{n} — clarification needed. Question posted. Moving on.` Single: `Paused: clarification needed … answer in the Questions tab, then re-run.` Otherwise continue to Step 5.5.

---

## Step 5.5 — Classify Difficulty (if unset)

Tasks queued from a plan item inherit that item's `difficulty`; manually-created tasks may have `task.difficulty` **null**. If null, assign a tier now so future runs route to the right model. Rubric:
- `light` — single-file or copy/config/content; mechanical; no schema changes; no ambiguity.
- `standard` — a typical feature: a few files, follows existing patterns, at most a small additive migration. **Default when unsure.**
- `heavy` — cross-cutting/architectural; new subsystems; schema redesign; ambiguous requirements; security-sensitive.

Persist with `update_task { task_id, fields: { difficulty: "{tier}" } }` (the field change is auto-logged). If already set, skip untouched.

---

## Step 6 — Create Implementation Plan

> **Checkpoint:** saved to the DB immediately — an interrupted run resumes from here.

Read the codebase first (use `ai_context` alongside the files, follow CLAUDE.md conventions). Write a plan covering: **Summary**, **Approach**, **Files to Create/Modify**, **Database Changes** (or "None"), **Sub-Skills Applied**, **Learnings Applied** (id + title from Step 4.5, or "None loaded"), **Testing**. Save it: `update_task { task_id, fields: { task_plan: "<the plan markdown>" } }` (auto-logs `plan_saved`). Print the plan for the user to review before code is written.

---

## Step 7 — Branch Setup

### Claim first (concurrency guard)
Call `claim_task { task_id }`. It atomically transitions `ready`/`changes_needed` → `in_progress` and logs the claim.
- **`{ claimed: true }`** → you hold the claim; continue.
- **`{ claimed: false, current_status }`** → `under_review`/`approved`/`done` → already handled, skip/stop; `in_progress` → another run owns it (batch: **skip**) or your own prior run was interrupted (single: **proceed** as resume — you effectively hold it).

(The resume-from-existing-PR path in Step 3.5 jumps to Step 11 and skips this claim.)

### `ready` — new branch (or resume existing)
Ensure a clean tree first; if dirty with unrelated work, `git stash push -u -m "pre-task-{task_number} WIP"` and tell the user. Then:
```bash
git fetch origin
git branch -a | grep "feature/task-{task_number}-"          # resume if it already exists (checkout + pull)
git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
git checkout -b feature/task-{task_number}-{short-slug}      # {short-slug} = 3–5 word kebab-case of the title
```
`log_task_history { action: "branch_created" }`.

### `changes_needed` — check out existing branch
```bash
BR=$(gh pr view {task.pr_number} --json headRefName -q '.headRefName')
git fetch origin && git checkout "$BR" && git pull origin "$BR"
```

---

## Step 8 — Implement

> Plan generation/revision live in `/generate-plan` and `/revise-plan`, not here — this skill only writes code (enforced by the Step 3 type guard).

Follow the plan. Read adjacent files and match existing patterns exactly. For DB changes: new numbered migration in `supabase/migrations/` + update `supabase/schema.sql`. Type-check with `npx tsc --noEmit` while developing; commit logical units (`npm run build` is the Step 9 gate).

**Sub-skills — invoke when applicable:** `/frontend-design`, `/react-best-practices`, `/web-design-guidelines`, `/composition-patterns`, `/ux-review`.

**Fabrio conventions:** dark mode only (`zinc-950` → `zinc-900` → `zinc-800`); success `emerald-400`, destructive `rose-400`; API routes follow `app/api/sites/` & `app/api/tasks/`; hooks follow the SWR pattern in `hooks/useSites.ts`; all DB changes need a numbered migration + schema.sql update.

---

## Step 9 — Build Check Before PR

Run `npm run build` in the repo you changed. **Do not create the PR if it fails.** Fix all issues, re-run `npx tsc --noEmit` then `npm run build` until green, commit the fixes, and `log_task_history { action: "build_fixed" }`. Only proceed at exit code 0.

---

## Step 10 — Create or Update PR

> **Checkpoint:** PR url/number saved immediately — an interrupted run resumes from here.

### `ready` — create PR (target `$BASE_BRANCH`)
```bash
git push -u origin feature/task-{task_number}-{short-slug}
gh pr create --base "$BASE_BRANCH" --title "Task #{task_number}: {task.title}" --body "$(cat <<'PRBODY'
## Task #{task_number} — {task.title}

**Type:** {task.type}  **Site:** {task.site_name}

### Summary
{task.feature_summary}

### Changes
{bullet list of files created/modified and what was done}

### Acceptance Criteria
{task.acceptance_criteria}

### Testing
{how to verify}

---
🤖 Implemented by AI via Fabrio `/feature-request`
PRBODY
)"
gh pr view --json url,number          # capture pr_url, pr_number
```
Save: `update_task { task_id, fields: { pr_url: "{pr_url}", pr_number: {pr_number} } }` (auto-logs `pr_linked`).

### `changes_needed` — push to existing PR
```bash
git push origin {branch_name}        # PR already exists; pr_url/pr_number unchanged
```

---

## Step 11 — Update Task Status

`update_task { task_id, fields: { status: "under_review" } }` (auto-logs the status change). Add a note if useful: `log_task_history { task_id, action: "ready_for_review", notes: "Implementation complete — PR #{pr_number} ready for review" }`.

---

## Step 11.5 — Retrospective

**Runs after every task (single + batch).** Record what this run taught you. Reflect:

1. Codebase surprises not in `ai_context` → `code_pattern`, scoped to this site.
2. What failed on first attempt → `pitfall`.
3. `changes_needed` runs — what the reviewer corrected, phrased as a rule → `review_feedback` (highest value; always record/reinforce one when you processed review comments).
4. Stylistic/product preference revealed → `preference`.
5. Workflow friction → `process`, usually portfolio-wide.

**Rules:** 0–3 learnings per run (zero is valid). Each is a generalizable rule, never a task recap. `title` ≤ 200 chars, `content` ≤ 2000 chars, actionable.

**Dedup:** compare each candidate to `loaded_learnings`. If it restates an existing one, `reinforce_learning { learning_id }` instead of inserting. **Insert new** with `record_learning { department: task.department, category, title, content, site_id: task.site_id (or omit for portfolio-wide), source_task_id: task.id }`.

**Always log** (even at 0): `log_task_history { task_id, action: "retrospective_saved", notes: "Recorded {N} learning(s), reinforced {M}" }`.

---

## Step 12 — Output Summary

**Single:**
```
✅ Task #{n} complete.
  Title: {title}   Site: {site}   Branch: feature/task-{n}-{slug}
  PR: {pr_url}     Status: under_review   Learnings: {N} recorded, {M} reinforced
Awaiting human review — once approved, /merge-task handles the rest.
```
**Batch:** a one-liner per task (`✅ #{n} — {title} → {pr_url}` / `⏭  #{n} — {title} → skipped ({reason})`), then `Batch complete. {N} processed.` with the list, noting ✅ tasks are under review and skipped ones resume on re-run once questions are answered.

---

## Error Recovery

On unexpected failure: `log_task_history { action: "error", notes: "Skill failed at step {N}: {msg}" }`; leave git clean (commit or stash in-progress work); report the step and what was attempted. **Batch:** log and continue (`⚠️  Task #{n} failed at Step {N}: {error}. Moving on.`). **Resume** a failed task by re-running — Step 3.5 skips completed work automatically.
