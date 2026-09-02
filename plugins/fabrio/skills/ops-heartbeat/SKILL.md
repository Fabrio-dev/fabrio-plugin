---
name: ops-heartbeat
description: "Renamed to $fabrio:run-due-jobs, which runs the recurring jobs that are due plus the weekly upkeep. Executing tasks moved to fabrio-runner. This shim forwards and will be removed in a future release."
---

# Ops Heartbeat → Run Due Jobs

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:run-due-jobs`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

This skill was renamed. It used to do two unrelated jobs: run due recurring work, and execute ready tasks. Task execution now belongs to **`fabrio-runner`**, a plain Node process that polls the queue continuously — so what is left is the recurring side, and `$fabrio:run-due-jobs` is what it's called.

**Forward immediately:** run `$fabrio:run-due-jobs`, passing `--weekly` through if it was given, and follow it in full. Do not do any of the work here.

`--chain` no longer exists — it grouped ready *tasks* into shared-branch chains, and tasks are the runner's loop now. If it was passed, say so once and continue without it.

Mention once, briefly, that `$fabrio:ops-heartbeat` is deprecated and `$fabrio:run-due-jobs` is the command from now on — and that anything scheduling this (cron, launchd, a cloud routine) should be updated. Don't belabor it.
