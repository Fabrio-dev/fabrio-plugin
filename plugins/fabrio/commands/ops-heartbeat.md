---
description: "Renamed to /fabrio:run-due-jobs, which runs the recurring jobs that are due plus the weekly upkeep. Executing tasks moved to fabrio-runner. This shim forwards and will be removed in a future release."
---

# Ops Heartbeat → Run Due Jobs

This skill was renamed. It used to do two unrelated jobs: run due recurring work, and execute ready tasks. Task execution now belongs to **`fabrio-runner`**, a plain Node process that polls the queue continuously — so what is left is the recurring side, and `/fabrio:run-due-jobs` is what it's called.

**Forward immediately:** run `/fabrio:run-due-jobs`, passing `--weekly` through if it was given, and follow it in full. Do not do any of the work here.

`--chain` no longer exists — it grouped ready *tasks* into shared-branch chains, and tasks are the runner's loop now. If it was passed, say so once and continue without it.

Mention once, briefly, that `/fabrio:ops-heartbeat` is deprecated and `/fabrio:run-due-jobs` is the command from now on — and that anything scheduling this (cron, launchd, a cloud routine) should be updated. Don't belabor it.
