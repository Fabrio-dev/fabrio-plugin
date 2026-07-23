---
description: "Proposes an updated version of a living plan based on completed work and accrued learnings. The human accepts or rejects the proposal in the UI."
---

# Revise Plan

Propose an evolved version of a living plan. This is the core of the "living plans" loop: the plan improves over time as work ships and learnings accrue. **This skill only proposes — it never mutates the live plan.** A human reviews the diff and accepts or rejects it at `/plans/{id}`.

**Invocation:** `/fabrio:revise-plan <plan_number>` — the short number shown as `#N` in Fabrio (not the UUID).

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials. The server scopes to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**. If the tools aren't available, stop and tell the user: create a key in **Settings → API keys** and run its **Connect command** (`claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"`), then restart Claude Code and re-invoke.

---

## Step 1 — Fetch the Plan + Current Items

Call `get_plan { plan_number, include_items: true }`. Store as `plan`; capture `plan.id` (the UUID). The plan is a cross-department **objective** (`plan.title`); each item carries its own `department`. It also returns `latest_accepted_revision` (used in Step 2). If null, tell the user the plan number wasn't found. If the plan has no items yet, stop and tell the user to run `/fabrio:generate-plan {plan_number}` first — there's nothing to revise.

---

## Step 2 — Determine What Changed Since the Last Revision

Use `plan.latest_accepted_revision.created_at` (from Step 1) as the cutoff. Fetch tasks for this site across **all** departments that moved since then:
`list_tasks { site_id: plan.site_id, updated_since: "{last_accepted_created_at}", order: "desc" }`

`done` tasks = initiatives that shipped (retire or build on them). `changes_needed` / repeated churn = friction worth addressing. Because the objective spans departments, watch for cross-department bottlenecks — e.g. content shipped but the design/dev work it depends on hasn't. If there is no prior accepted revision, review the site's task history broadly (`list_tasks { site_id: plan.site_id }`).

---

## Step 3 — Load Learnings & Sibling Sites

`list_learnings { site_id: plan.site_id, include_portfolio: true, statuses: ["active"], limit: 20 }`.

Also call `list_sites` so you know the account's other codebases. Existing items may already carry a `site_id` override (an initiative on a sibling repo, e.g. the public marketing site) — **carry it forward** in the proposed set, and set `site_id` on any new item that belongs to a different site than the plan's.

---

## Step 4 — Propose the Revision

Design an updated plan (goals + full item list) across all the departments the objective needs. Principles:
- **Keep** initiatives still relevant and not yet done.
- **Retire** initiatives whose work shipped (a matching `done` task) — don't re-propose completed work.
- **Add** new initiatives informed by what shipped, the goals, and the learnings — including in departments not yet represented if the objective now needs them.
- **Reprioritize** based on cross-department bottlenecks, what struggled, or what learnings flag as high-value.

The item list is the COMPLETE proposed set (it replaces the current items if accepted), each with `department`, `title`, `description`, `category`, `frequency`, `priority`, `difficulty`, `sort_order`, optional `depends_on` (the `sort_order` of a prerequisite item; auto-queue skips a dependent item until its prerequisite is `done`), optional `site_id` (a sibling site's id when the item belongs to a different codebase than the plan's site — carry existing overrides forward), and optional `kind`. Write a concise `change_summary` (what the human sees first).

**Generator jobs:** carry an existing generator item forward with its `kind: "generator"` and its `description` intact (its saved `job_plan` persists on the live item — a proposal doesn't touch it). Only propose a **new** generator when the objective genuinely needs recurring, source-driven work that files a ticket per issue (e.g. "keep beta bugs low"); set `kind: "generator"` and put a clear "how we gather work" account in the item's `description`, and note that the human runs `/fabrio:plan-job` on it after accepting to author the job plan.

`difficulty` is the effort tier (`light`/`standard`/`heavy`) that decides which model runs the item's tasks. **Carry each existing item's current difficulty forward** unless the revision genuinely re-scopes it; assign new items by the rubric (light = single-file/copy/config, no schema; standard = typical multi-file feature, default; heavy = cross-cutting/architectural/ambiguous/security-sensitive).

---

## Step 5 — Write the Proposed Revision (do NOT apply)

Call `propose_plan_revision`:
```
propose_plan_revision {
  plan_id: "{plan.id}",
  goals: "…updated goals…",
  target_audience: "…",
  items: [
    { department: "development", title: "…", description: "…", category: "infra", frequency: "one_time", priority: "high",   difficulty: "standard", sort_order: 0 },
    { department: "content",     title: "…", description: "…", category: "blog",  frequency: "monthly",  priority: "medium", difficulty: "light",    sort_order: 1 }
  ],
  change_summary: "Retired 2 shipped initiatives, added 3 (design + dev) informed by X; reprioritized Y."
}
```
The tool assigns the next revision number and status `proposed`. The live plan is untouched.

**Goals convention:** pass `goals`/`target_audience` only when proposing a change to them. To leave the plan's goals as-is, omit those fields (or set them to the current values) — the accept flow treats `null` as "no change" and won't overwrite live goals.

No retrospective — this skill wrote no code; the meaningful outcome is the human's accept/reject decision.

---

## Step 6 — Output Summary

```
📝 Proposed revision #{N} for {plan.site.name}

{change_summary}

  {current_count} current → {proposed_count} proposed initiatives

Review the diff and Accept/Reject at /plans/{plan.id}. Nothing changes until you accept.
```
