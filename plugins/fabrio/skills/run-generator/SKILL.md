---
name: run-generator
description: "Renamed to $fabrio:run-job, which runs every recurring job — not just ticket-filing ones. This shim forwards to it and will be removed in a future release."
---

# Run Generator → Run Job

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:run-due-jobs`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

This skill was renamed. It used to run only `generator` jobs; the runner now walks **every** recurring job's step tree, so "generator" no longer describes it.

**Forward immediately:** run `$fabrio:run-job <item_number>` with the same argument you were given, and follow it in full. Do not do any of the work here.

If no argument was given, say so and stop — `$fabrio:run-job` needs the job's `#N`.

Mention once, briefly, that `$fabrio:run-generator` is deprecated and `$fabrio:run-job` is the command from now on. Don't belabor it.
