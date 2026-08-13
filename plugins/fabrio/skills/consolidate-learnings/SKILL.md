---
name: consolidate-learnings
description: "Merges duplicate learnings, promotes recurring ones into site ai_context, and archives stale ones."
---

# Consolidate Learnings

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:ops-heartbeat`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

Keep the AI memory high-signal: merge duplicates, promote proven learnings into durable context, prune stale ones.

**Invocation:** `$fabrio:consolidate-learnings`

---

## Prerequisites

All data access is through the **`fabrio` MCP server** (`mcp__fabrio__*` tools) — **no Supabase CLI and no `SUPABASE_DB_URL` anymore.** The server scopes to the account whose API key is connected; connect or switch accounts with the connect command from **Fabrio → Settings → API keys**. If the tools aren't available, stop and tell the user: create a key in **Settings → API keys** and run its **Connect command** (`export FABRIO_API_KEY="fab_live_YOUR_KEY"\ncodex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`), then restart Codex and re-invoke.

---

## Step 1 — Fetch Active Learnings

Call `list_learnings { statuses: ["active"] }` (no department/site filter → the whole account). Group the results by `(site_id, department)`. If empty, output "No active learnings to consolidate." and stop.

---

## Step 2 — Merge Duplicates

Within each `(site_id, department)` group, identify near-duplicate learnings — two learnings that state the same rule in different words. **Judge by meaning, not string similarity.**

For each duplicate cluster, pick the **clearest, most actionable** phrasing as the survivor and call:
`merge_learnings { survivor_id, duplicate_ids: [ … ] }`
The server sums the cluster's `occurrence_count` onto the survivor and archives the duplicates. Be conservative: if two learnings are related but distinct rules, leave both.

---

## Step 3 — Promote Proven Learnings

### Site-scoped (`site_id` set, `occurrence_count >= 3`)

For each site with qualifying learnings, collect the new `active` learnings with `occurrence_count >= 3` and call:
`promote_learnings { site_id, learning_ids: [ … ] }`

The server marks them `promoted` **and** rewrites the site's `## Learned Conventions` section server-side — merging them with any already-promoted learnings and preserving all human-authored content outside that section. It returns `updated_section` (print it in the report). You do **not** fetch or rewrite `ai_context` yourself. Promoted learnings keep their rows as provenance but are excluded from workflow injection.

### Portfolio-wide (`site_id` null, `occurrence_count >= 3`)

There is still no server-side promotion path for portfolio learnings, so do **not** write them anywhere. Split them in the final report so the user can act on each:
- **Workspace-instruction candidates** — rules that hold for every site regardless of department (tooling, git provider, naming, company-wide policy). Their home is **Settings → AI instructions** (`accounts.ai_context`, 031).
- **Playbook candidates** — craft conventions belonging to one department (department playbook writing lands in a later phase).

---

## Step 4 — Prune Stale Learnings

Call `archive_stale_learnings { days: 90 }` — the server archives `active` learnings with `occurrence_count = 1` not seen in 90 days and returns the archived rows.

---

## Step 5 — Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Learnings consolidation complete.

  {site or "Portfolio"} × {department}: {N} merged, {N} promoted, {N} archived
  ...

Promoted into ai_context:
  • {site}: "{learning title}"
  ...

Workspace AI instruction candidates (paste into Settings → AI instructions):
  • "{learning title}" (seen {N}×)
  ...

Playbook candidates (portfolio-wide, review manually):
  • [{department}] "{learning title}" (seen {N}×)
  ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

For each site whose `ai_context` changed, also print the `updated_section` returned by `promote_learnings` so the user can eyeball it.
