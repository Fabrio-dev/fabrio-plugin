---
name: generate-plan
description: "Generates the initial cross-department initiatives for an empty objective plan (each item is assigned to development, design, marketing, or content)."
---

# Generate Plan

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:ops-heartbeat`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

Fill an empty objective plan with an initial set of initiatives spanning whatever departments the objective requires (content, design, development, marketing).

**Invocation:** `$fabrio:generate-plan <plan_number>` — the short number shown as `#N` in Fabrio (not the UUID).

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — no Supabase credentials, and **no** requirement that the Next.js app be running locally (the MCP server is the app). The server scopes to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**. If the tools aren't available, stop and tell the user: create a key in **Settings → API keys** and run its **Connect command** (`export FABRIO_API_KEY="fab_live_YOUR_KEY"\ncodex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`), then restart Codex and re-invoke.

---

## Step 1 — Fetch the Plan

Call `get_plan { plan_number, include_items: true, include_attachments: true }`. Store as `plan`; capture `plan.id` (the UUID) — the item-writing tools use it. The plan is a cross-department **objective** (`plan.title`, e.g. "Improve SEO") — no department on the plan itself; you assign a department to each item. If null, tell the user the plan number wasn't found. If the plan already has items, warn that `$fabrio:generate-plan` replaces them and suggest `$fabrio:revise-plan` for incremental updates — proceed only on a full regeneration.

### Resolve the plan's TARGET SITES

**A plan targets a set of sites, not one site.** Build that list now and store it as `targets` — everything downstream reads it:

- `plan.all_sites === true` → call `list_sites` and take every site whose `status` is `active`.
- otherwise → `plan.plan_sites[].site` (each carries `id`, `name`, `relative_path`, `live_url`, `description`, `ai_context`). If that array is empty, fall back to the single `plan.site`.

Do **not** use `plan.site` / `plan.site_id` as "the plan's site" — it is a legacy pointer at the *first* site and is `null` on an all-sites plan.

Say the target list back to the user before you generate, e.g. `Plan #12 "Improve SEO" targets 3 sites: Acme App, Acme Marketing, Acme Docs.`

---

## Step 2 — Load Workspace Context, Departments, Learnings & Sibling Sites

Call `get_account_context` first — the workspace's own `ai_context`: portfolio-wide rules (branch naming, company-wide code and content policy) that constrain every initiative you write, whatever site or department it lands on. Treat it as binding. It is the widest context layer; a department's `playbook` and a site's `ai_context` are narrower and win a direct conflict.

Then call `list_departments` — it returns each department's `slug`, `description` and `playbook`. Use the slugs as the valid set for `department` (never hardcode them) and let each `description` guide which items belong where. A department's `playbook`, when present, is how that department actually works here — respect it when shaping its initiatives.

Call `list_learnings { site_id: "{target.id}", include_portfolio: true, statuses: ["active"], limit: 20 }` **once per site in `targets`** (all departments — the objective spans several). Keep them grouped by site: a learning about Acme Marketing has no bearing on an initiative you pin to Acme App. Apply `preference`/`process` learnings to the plan's direction; treat `pitfall`/`review_feedback` as things to avoid. When generating an item for a department, weight that department's learnings most.

Also call `list_sites` to see codebases **outside** the plan's target set. A single product often spans repos — e.g. an app and its **public marketing site**. If the objective needs work in a sibling site the plan doesn't itself target, note that site's `id`; you can still set it as an item's `site_id` in Step 3, and the queued task targets that repo.

---

## Step 2.5 — Read Reference Documents

`plan.attachments` (from Step 1) may include brand guides, briefs, research, or screenshots. For each, pull the content into context via its `public_url` (public bucket):
- **Text docs** (`text/*`, `application/json`): `curl -s "{public_url}"` and read directly.
- **PDFs**: `curl -s "{public_url}" -o {scratch}/{filename}` then read with the Read tool.
- **Images**: download the same way, then read with the Read tool so you can see it.

Treat these as authoritative user context — they take precedence over generic assumptions. No attachments → skip.

---

## Step 3 — Generate Initiatives

Read the workspace instructions (from Step 2), then the context of **each site in `targets`** (`name`, `description` if present, `live_url`, `ai_context`) plus the objective (`plan.title`), `plan.goals`, `plan.target_audience`, and the reference documents. Decide **which departments the objective needs** and generate **6–12 initiatives spanning them**. Example — "Improve SEO" typically needs content (articles/copy), design (CWV, internal-linking UI), development (schema markup, sitemaps, performance), marketing (backlink outreach). Only include a department the objective genuinely needs.

### An item is an INITIATIVE, not a STAGE

Before you emit an item, ask: **would someone schedule this on its own, and would finishing it alone be worth something?** If it only makes sense as one phase of another item, it is **not an item** — it is a **step** of that item, and you must not emit it.

**The pipeline test (hard rule).** If two or more candidate items form a chain where each one's only consumer is the next — *fetch → analyse → classify → dedupe → decide → file* — that whole chain is **ONE** item. Emit the single **recurring** item, narrate the entire flow in its `description`, and do **not** emit the stages as siblings. `$fabrio:plan-job` later turns that description into the job's nested steps, including a `foreach` for the per-item work.

**`depends_on` sequences independent initiatives; it is not a workflow arrow.** Valid: "Publish the pricing page" depends on "Ship the pricing API" — two deliverables, two owners, each worth doing. Invalid: "Classify severity" depends on "Investigate errors" — two stages of one job.

**Never give a RECURRING item a `depends_on`.** A job performs its whole pipeline inside a single run, so a stage of that pipeline can never be a real prerequisite for it. (The server drops the edge anyway — it once produced a monitoring job displayed as *"blocked by: auto-create a ticket with the remediation plan"*, blocked by the very thing it does.) To hold a job until something else ships, set its start date rather than a dependency.

**Three signals you are about to make this mistake.** If any is true, collapse the chain into one item:
1. A `depends_on` chain is longer than two links.
2. Every link is `one_time` but the last link is recurring.
3. The last item's `description` restates what the earlier items do.

### Every initiative names ONE site

An objective can span several repos; **an initiative almost never does.** Before you emit an item, ask: **which one codebase does this actually land in?** Read the target sites' `name`, `description`, `live_url` and `ai_context` — that is what tells you whether "rewrite the pricing copy" belongs to the marketing site or the app. Set that site's `id` as the item's `site_id`. The item then files **one** task, in that repo.

**`all_plan_sites: true` is the rare case.** Use it only when the item is genuinely *the same job repeated per repo* — add a `robots.txt` to each, bump a shared dependency everywhere, apply one security header across the portfolio. The item then fans out into one task per targeted site.

**The reuse test.** Would the finished work on site A be usable as-is on site B? If no, it is not one fan-out item — it is **separate items, one per site**, each with its own `site_id` and its own title. Three sites needing three different landing pages is three initiatives, not one.

**On a plan targeting more than one site, an item carrying neither `site_id` nor `all_plan_sites: true` is REJECTED** by `replace_plan_items`, with the offending titles named. There is no accidental default: decide for every item. (On a single-site plan neither field is needed — the one site is unambiguous.)

**Sizing.** 6–12 initiatives is the budget for the *plan*, not per site. A plan across 3 sites is still 6–12 items total; pin them where the work lands rather than mirroring the same list three times.

**Self-check before Step 4.** Read your item list back. For each recurring item, ask: *does any other item describe a stage of this one?* If so, delete that item and fold its detail into the recurring item's `description`. Then confirm no `depends_on` chain exceeds two links and no recurring item has one at all.

Each initiative:
- `department` — the owning department's `slug`, from `list_departments` (call it rather than assuming the set; also read each `description` to place items well). Today: `development` | `design` | `marketing` | `content`.
- `title` (≤200 chars)
- `description` (one or two sentences of actionable detail)
- `category` (free-form lowercase slug — see suggestions)
- `frequency` — `one_time` | `weekly` | `biweekly` | `monthly` | `quarterly`
- `priority` — `high` | `medium` | `low`
- `difficulty` — `light` | `standard` | `heavy` (effort tier for model routing):
  - `light` — single-file / copy / config / content; mechanical; no schema changes. Most `marketing`/`content`.
  - `standard` — a typical feature: a few files, existing patterns, small additive migration at most. **Default.**
  - `heavy` — cross-cutting/architectural; new subsystems; schema redesign; ambiguous; security-sensitive.
- `execution_mode` — **how the finished thing lands.** Independent of `department`; set it whenever you can tell, and omit only when genuinely unclear (the executor then classifies it on the first run):
  - `repo` — the deliverable is files in a site repo, so it ships as a branch + PR. A blog post in `content/posts/`, landing copy, SEO metadata, an email template, any code or design change. **Prefer this when a repo could hold it** — it inherits review, history and deploy for free.
  - `artifact` — a document with no repo home, useful on its own: a keyword/topic backlog, a content calendar, an ad-copy set, a competitive brief, a monthly report, an outreach target list.
  - `external` — the work *is* an action on a third-party system: publishing to a social channel, sending a campaign, launching or adjusting paid ads, contacting people. Fabrio prepares a ready-to-execute package; **a human always performs the action.**

  Do **not** infer it from the department: a `marketing` item can be `repo` (a landing page) and a `development` item can be `artifact` (an architecture proposal). Think about where the output lives, not who owns it.
- `sort_order` — integer, ascending
- `depends_on` — **optional** integer: the `sort_order` of a prerequisite item. Sequences genuinely **independent** initiatives; auto-queue skips a dependent item until its prerequisite is `done`. Omit for independent items, and never set it on a recurring one (see the pipeline test above).

**For every RECURRING item, make the `description` a clear, natural-language account of the WHOLE flow** — the source (API, ticket system, MCP, analytics), how to pick what to work on, and what it produces (e.g. "Each week, pull open bugs from the team's tracker, take the top 5 by severity, and file a ticket for each"). `$fabrio:plan-job` later compiles that description into the job's nested step tree — steps that plan the work and file a task, never steps that do the work themselves. **Its stages are steps of this one item — never sibling items.**
- `site_id` — **which site this initiative lands in**, an `id` from `targets` (or from `list_sites` for a sibling repo the plan doesn't target). The queued task is created against that site's repo, so cross-site sequencing works — e.g. a public-site page that `depends_on` an app feature. **Set this on nearly every item.**
- `all_plan_sites` — `true` **instead of** `site_id`, for work that must be repeated in every targeted repo. Required to fan out on a multi-site plan; unnecessary on a single-site one. See *Every initiative names ONE site* above.

**Category suggestions** (hints): marketing → seo, social, email, content, paid · development → feature, refactor, infra, bugfix, performance · design → visual, ux, design-system, branding · content → blog, landing, docs, email, social.

---

## Step 4 — Save Items (records accepted revision #1)

Call `replace_plan_items { plan_id: plan.id, items: [ … ], change_summary: "Initial generation" }`. This bulk-replaces the plan's items, marks the plan `active`, and records an **accepted** revision snapshot. Uses `plan.id` (the UUID), not the plan number.

Most items carry a `site_id`; the fan-out one carries `all_plan_sites: true`. If the call is rejected for an item with no target site, that is the guard from Step 3 — go back, decide where that initiative lands, and re-send. Do not blanket-set `all_plan_sites` to get past it.

```
replace_plan_items {
  plan_id: "{plan.id}",
  items: [
    { department: "content",     title: "…", description: "…", category: "blog",   frequency: "monthly",  priority: "high",   difficulty: "light",    execution_mode: "repo",     site_id: "{marketing_site_id}", sort_order: 0 },
    { department: "development", title: "…", description: "…", category: "infra",  frequency: "one_time", priority: "medium", difficulty: "standard", execution_mode: "repo",     site_id: "{app_site_id}",       sort_order: 1 },
    { department: "marketing",   title: "…", description: "…", category: "social", frequency: "weekly",   priority: "medium", difficulty: "light",    execution_mode: "external", site_id: "{marketing_site_id}", sort_order: 2 },
    { department: "development", title: "Add security headers to every site", description: "…", category: "infra", frequency: "one_time", priority: "medium", difficulty: "light", execution_mode: "repo", all_plan_sites: true, sort_order: 3 }
  ],
  change_summary: "Initial generation"
}
```

---

## Step 5 — Retrospective

Record 0–3 generalizable learnings (same rules as feature-request Step 11.5: recording nothing is valid; no task recaps). Reflect: what about a target site's positioning/audience or codebase wasn't in the workspace or site `ai_context`? Did generating this plan reveal a durable preference? Dedup against Step 2 — `reinforce_learning { learning_id }` rather than inserting a restatement. Each learning carries the department it concerns and **the site it concerns** — the one whose context it is about, not the plan's legacy pointer: `record_learning { site_id: "{site_it_concerns}", department: "{department_it_concerns}", category, title, content }`. Pass `site_id: null` for something true of the whole portfolio.

---

## Step 6 — Output Summary

```
✅ Plan generated: "{plan.title}" — {S} site(s): {target site names}

{N} initiatives across {D} departments (accepted as revision #1):
• [content]     {title} — {site name} · {frequency} · {execution_mode}
• [design]      {title} — {site name} · {frequency} · {execution_mode}
• [development] {title} — all {S} sites · {frequency} · {execution_mode}
...

Per site: {site name} {n}, {site name} {n}, all sites {n}

Review and queue tasks at /plans/{plan.id}. To evolve the plan later, run $fabrio:revise-plan {plan_number}.
```

Every **recurring** initiative needs its steps authored before it can run — tell the user: `$fabrio:plan-job {item_number}` (the job's `#N`, shown on the job in the plan UI). The ops heartbeat also does this automatically for any job with a description and no procedure.

If any initiative is `execution_mode: "external"`, say so plainly — Fabrio will prepare a ready-to-execute package for each, but **the human performs the action**; those items never publish, send or spend on their own.
