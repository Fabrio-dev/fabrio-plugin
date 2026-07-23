---
description: "Generates the initial cross-department initiatives for an empty objective plan (each item is assigned to development, design, marketing, or content)."
---

# Generate Plan

Fill an empty objective plan with an initial set of initiatives spanning whatever departments the objective requires (content, design, development, marketing).

**Invocation:** `/fabrio:generate-plan <plan_number>` — the short number shown as `#N` in Fabrio (not the UUID).

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials, and **no** requirement that the Next.js app be running locally (the MCP server is the app). The server scopes to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**. If the tools aren't available, stop and tell the user: create a key in **Settings → API keys** and run its **Connect command** (`claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"`), then restart Claude Code and re-invoke.

---

## Step 1 — Fetch the Plan

Call `get_plan { plan_number, include_items: true, include_attachments: true }`. Store as `plan`; capture `plan.id` (the UUID) — the item-writing tools use it. The plan is a cross-department **objective** (`plan.title`, e.g. "Improve SEO") — no department on the plan itself; you assign a department to each item. If null, tell the user the plan number wasn't found. If the plan already has items, warn that `/fabrio:generate-plan` replaces them and suggest `/fabrio:revise-plan` for incremental updates — proceed only on a full regeneration.

---

## Step 2 — Load Learnings & Sibling Sites

Call `list_learnings { site_id: plan.site_id, include_portfolio: true, statuses: ["active"], limit: 20 }` (all departments — the objective spans several). Apply `preference`/`process` learnings to the plan's direction; treat `pitfall`/`review_feedback` as things to avoid. When generating an item for a department, weight that department's learnings most.

Also call `list_sites` to see the account's other codebases. A single product often spans repos — e.g. an app and its **public marketing site**. If the objective needs work in a sibling site (marketing/content items usually land on the public site, not the app), note that site's `id`; you'll set it as the item's `site_id` in Step 3 so the queued task targets the right repo.

---

## Step 2.5 — Read Reference Documents

`plan.attachments` (from Step 1) may include brand guides, briefs, research, or screenshots. For each, pull the content into context via its `public_url` (public bucket):
- **Text docs** (`text/*`, `application/json`): `curl -s "{public_url}"` and read directly.
- **PDFs**: `curl -s "{public_url}" -o {scratch}/{filename}` then read with the Read tool.
- **Images**: download the same way, then read with the Read tool so you can see it.

Treat these as authoritative user context — they take precedence over generic assumptions. No attachments → skip.

---

## Step 3 — Generate Initiatives

Read the site context (`plan.site.name`, `description` if present, `live_url`, `ai_context`) plus the objective (`plan.title`), `plan.goals`, `plan.target_audience`, and the reference documents. Decide **which departments the objective needs** and generate **6–12 initiatives spanning them**. Example — "Improve SEO" typically needs content (articles/copy), design (CWV, internal-linking UI), development (schema markup, sitemaps, performance), marketing (backlink outreach). Only include a department the objective genuinely needs.

Each initiative:
- `department` — `development` | `design` | `marketing` | `content`
- `title` (≤200 chars)
- `description` (one or two sentences of actionable detail)
- `category` (free-form lowercase slug — see suggestions)
- `frequency` — `one_time` | `weekly` | `biweekly` | `monthly` | `quarterly`
- `priority` — `high` | `medium` | `low`
- `difficulty` — `light` | `standard` | `heavy` (effort tier for model routing):
  - `light` — single-file / copy / config / content; mechanical; no schema changes. Most `marketing`/`content`.
  - `standard` — a typical feature: a few files, existing patterns, small additive migration at most. **Default.**
  - `heavy` — cross-cutting/architectural; new subsystems; schema redesign; ambiguous; security-sensitive.
- `sort_order` — integer, ascending
- `kind` — **optional**: `execution` (default) or `generator`. Use `generator` **only** for a recurring initiative that gathers work from a source and files a ticket per issue (e.g. "weekly bug triage → file a ticket per bug", "review churned users → outreach tasks") rather than being a single unit of work. A generator must be recurring (`frequency != one_time`). For a generator, make the **`description`** a clear, natural-language account of how the job gathers work — the source (API, ticket system, MCP, analytics) and how to pick items (e.g. "Each week, pull open bugs from the team's tracker, take the top 5 by severity, and file a ticket for each"). `/fabrio:plan-job` later compiles that description into the saved job plan (and may ask the human a clarifying question first).
- `depends_on` — **optional** integer: the `sort_order` of a prerequisite item. Sequences cross-department work; auto-queue skips a dependent item until its prerequisite is `done`. Omit for independent items.
- `site_id` — **optional**: a sibling site's `id` (from Step 2's `list_sites`) when this initiative belongs to a **different** codebase than the plan's site — e.g. a marketing item for the public site while the plan is on the app. Omit to inherit the plan's site (the common case). The queued task is created against this site's repo, so cross-site sequencing works (e.g. a public-site page that `depends_on` an app feature).

**Category suggestions** (hints): marketing → seo, social, email, content, paid · development → feature, refactor, infra, bugfix, performance · design → visual, ux, design-system, branding · content → blog, landing, docs, email, social.

---

## Step 4 — Save Items (records accepted revision #1)

Call `replace_plan_items { plan_id: plan.id, items: [ … ], change_summary: "Initial generation" }`. This bulk-replaces the plan's items, marks the plan `active`, and records an **accepted** revision snapshot. Uses `plan.id` (the UUID), not the plan number.

```
replace_plan_items {
  plan_id: "{plan.id}",
  items: [
    { department: "content",     title: "…", description: "…", category: "blog", frequency: "monthly",  priority: "high",   difficulty: "light",    sort_order: 0 },
    { department: "development", title: "…", description: "…", category: "infra", frequency: "one_time", priority: "medium", difficulty: "standard", sort_order: 1 }
  ],
  change_summary: "Initial generation"
}
```

---

## Step 5 — Retrospective

Record 0–3 generalizable learnings (same rules as feature-request Step 11.5: recording nothing is valid; no task recaps). Reflect: what about this site's positioning/audience or codebase wasn't in `ai_context`? Did generating this plan reveal a durable preference? Dedup against Step 2 — `reinforce_learning { learning_id }` rather than inserting a restatement. Each learning carries the department it concerns, scoped to the site: `record_learning { site_id: plan.site_id, department: "{department_it_concerns}", category, title, content }`.

---

## Step 6 — Output Summary

```
✅ Plan generated: "{plan.title}" for {plan.site.name}

{N} initiatives across {D} departments (accepted as revision #1):
• [content]     {title} — {frequency}
• [design]      {title} — {frequency}
• [development] {title} — {frequency}
...

Review and queue tasks at /plans/{plan.id}. To evolve the plan later, run /fabrio:revise-plan {plan_number}.
```

If any initiative was created as a **generator** (`kind: "generator"`), tell the user it needs a job plan before it can run: `/fabrio:plan-job {item_number}` (the job's `#N`, shown on the job in the plan UI).
