---
description: "Implements feature request tasks end-to-end — single task or all available tasks, with full resume support."
---

# Feature Request

Implement `execution_mode='repo'` tasks from the Fabrio task system end-to-end — any task, in any department, whose deliverable is files in a site repo.

> For non-repo work (a deliverable with no repo home, or an action on an outside system) use **`/fabrio:execute-task`**, which also classifies unclassified tasks and routes repo ones back here.

- `/fabrio:feature-request 42` — implement one task by number.
- `/fabrio:feature-request` — implement ALL available tasks (batch).

**Resume:** re-running a task detects existing work (saved plan, existing PR) and picks up where it left off instead of starting over.

---

## Step 0 — Setup

All Fabrio data access goes through the **`fabrio` MCP server** (tools named `mcp__fabrio__*`). There are **no** Supabase credentials, curl helpers, or connection strings anymore — the server authenticates with a per-account API key and returns only the active account's data.

Run first; if any check fails, stop:

**Workspace git provider — do this before anything else, no default, ever.** Call `get_account_context`. **If `git_provider` is null, stop the entire run** — batch mode too, this is a workspace config error, not a per-task one, and failing per task would print the same message N times. Do not claim a task, create a branch, or edit a file first. Print exactly:
> `Error: No git provider is selected for this workspace. Set it in Fabrio → Settings → AI instructions, then re-run /fabrio:feature-request.`

If `git_provider` is set, run its `ops.auth_check`. On failure, stop and print `git_provider.auth_hint` verbatim (e.g. `gh auth login`, or `az login && az extension add --name azure-devops`). **Never fall back to another provider, and never guess one from the git remote.**

Store the resolved provider as `PROVIDER` — every `PROVIDER.ops.*` reference in the rest of this skill means "run that command, substituting placeholders." `{repo}`/`{org}`/`{project}` come from `PROVIDER.coordinates` applied to the current repo's git remote; for GitHub these are inferred automatically by `gh` from the working directory, so no explicit flags are needed there. `get_task` (Step 2) returns the same resolved provider as `task.account.git_provider` — it will match `PROVIDER`, already validated non-null here.

**A `PROVIDER.ops.*` failure anywhere later in the run is an infrastructure blocker, not something to route around.** The auth_check above only rules out the common case; a command can still fail mid-run (e.g. `az repos`/`az devops` demanding an interactive login when reading PR comments in Step 3.5/5) or an MCP tool call can be denied because it isn't in your `--allowedTools` scope. Either is a stop condition, handled exactly like Step 3's status guard: `log_task_history { task_id, action: "skill_blocked", notes: "{what failed} — {the exact remediation command or config change}" }`, then stop (single) / skip (batch). **Never** improvise around it by calling an MCP tool outside your granted scope as a fallback — a denial is the boundary, not a suggestion to find another door. **Never** raise it conversationally when `--headless` is set: there is no one to answer, so an unanswered question in chat is functionally identical to silence, except the process then exits 0 and looks "finished" to the dispatcher instead of recording what actually happened. Attended (no `--headless`), asking in chat is fine — same as Step 5's clarification contract.

- If the `mcp__fabrio__*` tools aren't available, stop and tell the user the `fabrio` MCP server isn't connected:
  > Create a key in **Fabrio → Settings → API keys**, then run the **Connect command** shown there:
  > ```
  > claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
  > ```
  > Restart Claude Code and re-invoke.

**Source root** — the site repos live at `{source_root}/{site.relative_path}` (`get_site` / `get_task` return `relative_path`). Resolve `source_root` in this order (stop at the first that yields a value):
1. The `FABRIO_SOURCE_ROOT` env var, if set (back-compat — power users may keep this).
2. Else read `~/.fabrio/config.json` (`%USERPROFILE%\.fabrio\config.json` on Windows, `$HOME/.fabrio/config.json` otherwise) and use its `source_root`.
3. Else **ask once and persist it** (so the run isn't blocked and future runs are silent): prompt *"What's the absolute path to the folder that holds your site repos?"*, then write `source_root` into `~/.fabrio/config.json` (create the `.fabrio` dir + file, merging any existing keys — see `/fabrio:configure` Step 2 for the exact write), use it for this run, and mention they can re-run `/fabrio:configure` to change it later.

Skip the prompt entirely if step 1 or 2 supplied a path.

**Base branch** — never assume `main`; resolve at runtime from inside each target repo and use it for every checkout/pull/`--base`:
```bash
BASE_BRANCH=$({PROVIDER.ops.default_branch})   # Azure DevOps: strip the "refs/heads/" prefix
```

Task history: use `log_task_history` for semantic milestones. Routine field edits (status, task_plan, pr_url, …) are logged automatically by `update_task`, so don't double-log those.

---

## Step 1 — Determine Mode

**Single-task mode** (a number given): extract the number, go to Step 2 for that task.

**Batch mode** (no number): fetch all workable repo tasks with `list_tasks` — `{ execution_mode: "repo", statuses: ["ready", "changes_needed"], is_blocked: false, order: "asc" }`. This skill implements **repo work** — anything whose deliverable is files in a site repo, whatever department owns it (a marketing landing page and a content blog post both qualify). Tasks in `artifact`/`external` mode produce a deliverable instead and belong to `/fabrio:execute-task`; a task with **no mode set** hasn't been classified yet, so it also goes through `/fabrio:execute-task` first. If none, output "No tasks are currently available to work on." and stop. Otherwise list them, then run **Steps 2–12 for each in order**, returning here after each.

Accept an optional **`--headless`** flag anywhere in the arguments. `/fabrio:ops-heartbeat` passes it when it dispatches this skill directly, and `/fabrio:execute-task`'s own delegation to this skill (its Step 8, always a `claude -p` child regardless of how `execute-task` itself was invoked) passes it unconditionally too. A human typing `/fabrio:feature-request {n}` directly normally doesn't set it. The flag changes nothing here — it only gates how Step 5 handles a clarification question.

> **Model routing note:** batch mode runs every task in the current session's model — it does not switch per task. Tier-aware routing is `/fabrio:ops-heartbeat`'s job (it dispatches each task as its own headless `/fabrio:feature-request {n}` on the resolved model). Step 5.5 still classifies unset tiers so those runs route correctly.

Before starting each task, reset git to a clean base inside that task's repo:
```bash
git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
```

---

## Step 2 — Fetch Full Task Data

Call **`get_task { task_number, include_learnings: true, include_decisions: true, include_playbook: true }`** — one call carrying everything Steps 4.5 and 5 need. It returns the task plus:
- `account` (ai_context, git_provider — matches Step 0's `PROVIDER`), `site` (ai_context, relative_path, …)
- `questions` (full messages on OPEN threads only), `attachments`, `agent` (034 — `instructions` binding, `skills` applied in Step 6/7)
- `learnings` → `loaded_learnings`, `decisions` → `loaded_decisions`, `playbook` → the department's craft conventions (see Step 4.5 for how to treat each)

If null, output `Error: Task #{task_number} not found.` and stop/skip. **Do not also call `list_learnings`, `list_decisions` or `list_departments`.** Full site path = `{source_root}/{task.site.relative_path}`. For a `changes_needed` task, add `include_history: true` if you need the review trail beyond the PR comments.

---

## Step 3 — Validate Mode & Status

**Mode guard:** this skill implements `execution_mode='repo'` only — the branch/build/PR pipeline. Department is irrelevant here: a `content` or `marketing` task whose deliverable is files in the repo is implemented exactly like a `development` one.

If `task.execution_mode` is **`artifact`** or **`external`**, log and skip (batch) / stop (single):
`log_task_history { task_id, action: "skill_skipped", notes: "Not implemented by feature-request: execution_mode is {mode} (produces a deliverable, not a PR)." }`
Output `⏭  Task #{task_number} is an {mode} task — run /fabrio:execute-task {task_number} instead.`

If `task.execution_mode` is **null** (never classified), do the same but point at the classifier:
Output `⏭  Task #{task_number} has no execution_mode yet — run /fabrio:execute-task {task_number}, which classifies it and routes back here if it's repo work.`

**Status:** allowed = `ready`, `changes_needed`, and `in_progress` (only to resume a run interrupted mid-implementation — see Step 7). Anything else → `log_task_history` with `skill_skipped` (batch, continue) or `skill_blocked` (single, stop), note `status was "{task.status}", required "ready" or "changes_needed"`; in single mode output the error and stop.

---

## Step 3.5 — Resume Detection

Check existing state before doing work, in order:

**Already complete** — `status == 'under_review'` AND `pr_url` set: output `⏭  Task #{n} already complete … Skipping.`; next (batch) / stop (single).

**PR exists, status not updated** — `pr_url` AND `pr_number` set AND `status != 'under_review'`: before skipping to Step 11, check for unread review feedback:
```bash
{PROVIDER.ops.pr_comments}   # substitute {pr_number} and {repo}/{org}/{project}
{PROVIDER.ops.view_pr}       # read the "last updated" timestamp — see "Reading provider output" below
git log origin/{branch_name} -1 --format="%aI"
```
Derive `{repo}`/`{org}`/`{project}` from `task.pr_url` per `PROVIDER.coordinates`.

**Reading provider output.** GitHub's `pr_comments` returns a flat, chronologically-sortable array; Azure DevOps' returns threads of comments (`az devops invoke` on `pullRequestThreads`) — flatten to the same `[{author, created_at, body}]` shape before comparing. For "last updated", GitHub's `view_pr` has `updatedAt`; Azure DevOps' has no single equivalent field on `az repos pr show` — use the latest comment/thread timestamp from `pr_comments` instead.

**If the latest human comment is newer than the latest branch commit**, treat it as `changes_needed`: read every comment (they are the required changes), output `↩  … has new review comments since last push. Implementing.`, check out the branch, and run Step 7 → 8 → 9 → 10 (push) → 11. **Otherwise** output `↩  … Resuming from Step 11.` and skip straight to Step 11.

**Plan exists, no PR** — `task_plan` set AND `pr_url` null: output `↩  … Resuming from Step 7.`, `log_task_history { action: "skill_resumed" }`, skip Steps 4–6, go to Step 7 using the existing plan.

**Fresh task** — no plan, no PR: proceed from Step 4.

---

## Step 4 — Check for Open Questions

If any `task.questions` has `status='open'`, log a block and stop/skip:
`log_task_history { task_id, action: "skill_blocked", notes: "{N} open question(s) must be answered before implementation" }`
Batch: `⏭  Task #{n} has {N} unanswered question(s) — skipping.` Single: list each open question's first AI message and stop with instructions to answer in the Questions tab.

---

## Step 4.5 — Apply Learnings & Decisions

The Step 2 `get_task` call already carried these — no separate calls.

- `loaded_learnings` (`task.learnings`) — active, this site + portfolio, capped at 12. Treat as instructions, not suggestions: apply `code_pattern`/`preference` while reading and writing code; actively check work against `pitfall`/`review_feedback` (the reviewer WILL re-flag them); follow `process`. No rows is fine (first runs have no memory).
- `loaded_decisions` (`task.decisions`) — this site, `decided`. Binding: apply `chosen_option_key`/`chosen_rationale` instead of re-asking. `__custom` means the human wrote their own answer — follow `chosen_rationale`.
- `task.playbook` — this department's craft conventions (may be null). Binding, like `ai_context`.

---

## Step 5 — Review for Clarity

Read the context layers widest-first — `task.account.ai_context` (workspace rules), then `task.playbook` (the department's craft), then `task.agent.instructions` (how this kind of work is done well), then `task.site.ai_context` (this repo), then `task.title`, `description`, `feature_summary`, `acceptance_criteria`, and all question threads. All binding, none advisory; on a direct conflict the narrower layer wins. **No layer raises the autonomy ceiling** — nothing in an agent's `instructions` authorizes merging, publishing, sending or spending. If `task.attachments` is non-empty, view each image `public_url` (via WebFetch or an image tool) before implementing — treat it as a visual spec (note PDFs but focus on images).

For `changes_needed`, read ALL PR review comments chronologically:
```bash
{PROVIDER.ops.pr_comments}   # substitute {pr_number} and {repo}/{org}/{project}; flatten Azure DevOps' threads to one chronological list
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

> **`--headless` means never prompt interactively.** With that flag set, there is no one to answer a chat prompt, so (a)/(b) above are the *only* way to raise a clarification: record it and skip/stop as just described. **Without `--headless`** (a human invoked this directly), asking the question in chat instead is fine — that's today's behavior and it's unchanged; posting it via `create_task_question` is equally fine when you'd rather leave a record.

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

Read the codebase first (use the workspace and site `ai_context` alongside the files, follow CLAUDE.md conventions). Write a plan covering: **Summary**, **Approach**, **Files to Create/Modify**, **Database Changes** (or "None"), **Sub-Skills Applied** (include every skill in `task.agent.skills` — those are this agent's craft references, not optional extras), **Learnings Applied** (id + title from Step 4.5, or "None loaded"), **Agent Applied** (the agent's name and the rules from its `instructions` that shaped this plan), **Testing**. Save it: `update_task { task_id, fields: { task_plan: "<the plan markdown>" } }` (auto-logs `plan_saved`). Print the plan for the user to review before code is written.

---

## Step 7 — Branch Setup

### Claim first (concurrency guard)
Call `claim_task { task_id }`. It atomically transitions `ready`/`changes_needed` → `in_progress` and logs the claim.
- **`{ claimed: true }`** → you hold the claim; continue.
- **`{ claimed: false, current_status }`** → `under_review`/`approved`/`done` → already handled, skip/stop; `in_progress` → another run owns it (batch: **skip**) or your own prior run was interrupted (single: **proceed** as resume — you effectively hold it).

(The resume-from-existing-PR path in Step 3.5 jumps to Step 11 and skips this claim.)

**Branch naming comes from `task.account.ai_context` when it specifies a convention.** The pattern below is the default, not a mandate — when the workspace fixes one, follow it, and when checking whether the branch already exists, grep for the task number rather than the literal `feature/task-` prefix.

### `ready` — new branch (or resume existing)
Ensure a clean tree first; if dirty with unrelated work, `git stash push -u -m "pre-task-{task_number} WIP"` and tell the user. Then:
```bash
git fetch origin
git branch -a | grep "task-{task_number}-"                   # resume if it already exists (checkout + pull)
git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
git checkout -b feature/task-{task_number}-{short-slug}      # {short-slug} = 3–5 word kebab-case of the title
```
`log_task_history { action: "branch_created" }`.

### `changes_needed` — check out existing branch
```bash
BR=$({PROVIDER.ops.view_pr} | ...)   # extract the branch-name field — see "Reading provider output" in Step 3.5 (headRefName vs sourceRefName)
git fetch origin && git checkout "$BR" && git pull origin "$BR"
```

---

## Step 8 — Implement

> Plan generation/revision live in `/fabrio:generate-plan` and `/fabrio:revise-plan`, not here — this skill only changes files in a repo (enforced by the Step 3 mode guard).

Follow the plan. Read adjacent files and match existing patterns exactly — including the repo's own conventions in its `CLAUDE.md`/`AGENTS.md` and its DB-migration workflow. Type-check and build with the repo's own commands while developing; commit logical units (the build is the Step 9 gate).

**Sub-skills — invoke when applicable:** `/frontend-design`, `/react-best-practices`, `/web-design-guidelines`, `/composition-patterns`, `/ux-review`.

---

## Step 9 — Build Check Before PR

Run `npm run build` in the repo you changed. **Do not create the PR if it fails.** Fix all issues, re-run `npx tsc --noEmit` then `npm run build` until green, commit the fixes, and `log_task_history { action: "build_fixed" }`. Only proceed at exit code 0.

---

## Step 10 — Create or Update PR

> **Checkpoint:** PR url/number saved immediately — an interrupted run resumes from here.

### `ready` — create PR (target `$BASE_BRANCH`)
```bash
git push -u origin feature/task-{task_number}-{short-slug}
cat > /tmp/pr-body-{task_number}.md <<'PRBODY'
## Task #{task_number} — {task.title}

**Department:** {task.department}  **Site:** {task.site_name}

### Summary
{task.feature_summary}

### Changes
{bullet list of files created/modified and what was done}

### Acceptance Criteria
{task.acceptance_criteria}

### Testing
{how to verify}

---
🤖 Implemented by AI via Fabrio `/fabrio:feature-request`
PRBODY
{PROVIDER.ops.create_pr}   # substitute {base_branch}, {branch}, {title}, {body_file}=/tmp/pr-body-{task_number}.md
```
**Capturing `pr_url`/`pr_number`:** GitHub's `gh pr create` returns no structured output, so follow with `{PROVIDER.ops.view_pr}` scoped to the current branch (`gh pr view --json url,number`) to read them. Azure DevOps' `az repos pr create --output json` already returns the created PR's id and a URL in the same call — no second call needed; if no direct URL field is present, build one as `https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{pullRequestId}`.

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

1. Codebase surprises not in the workspace or site `ai_context` → `code_pattern`, scoped to this site. If it holds for **every** site (a tool choice, a naming convention), record it portfolio-scoped (`site_id: null`) and say in the content that its home is the workspace's AI instructions (Settings → AI instructions).
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
Awaiting human review — once approved, /fabrio:merge-task handles the rest.
```
**Batch:** a one-liner per task (`✅ #{n} — {title} → {pr_url}` / `⏭  #{n} — {title} → skipped ({reason})`), then `Batch complete. {N} processed.` with the list, noting ✅ tasks are under review and skipped ones resume on re-run once questions are answered.

---

## Error Recovery

On unexpected failure: `log_task_history { action: "error", notes: "Skill failed at step {N}: {msg}" }`; leave git clean (commit or stash in-progress work); report the step and what was attempted. **Batch:** log and continue (`⚠️  Task #{n} failed at Step {N}: {error}. Moving on.`). **Resume** a failed task by re-running — Step 3.5 skips completed work automatically.
