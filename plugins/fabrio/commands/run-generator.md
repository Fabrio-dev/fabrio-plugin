---
description: "Renamed to /fabrio:run-job, which runs every recurring job — not just ticket-filing ones. This shim forwards to it and will be removed in a future release."
---

# Run Generator → Run Job

This skill was renamed. It used to run only `generator` jobs; the runner now walks **every** recurring job's step tree, so "generator" no longer describes it.

**Forward immediately:** run `/fabrio:run-job <item_number>` with the same argument you were given, and follow it in full. Do not do any of the work here.

If no argument was given, say so and stop — `/fabrio:run-job` needs the job's `#N`.

Mention once, briefly, that `/fabrio:run-generator` is deprecated and `/fabrio:run-job` is the command from now on. Don't belabor it.
