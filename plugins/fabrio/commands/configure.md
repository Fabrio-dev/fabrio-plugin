---
description: "One-time per-machine setup — saves the source root (folder holding your site repos) to ~/.fabrio/config.json so the workflow skills can find your code."
---

# Configure

One-time, per-machine setup. The workflow skills that touch code (`/fabrio:feature-request`, `/fabrio:merge-task`) build each repo's path as `{source_root}/{site.relative_path}`. This command saves your **source root** — the absolute path to the folder that holds your site repos — to `~/.fabrio/config.json` so those skills find your code without any env var.

**Invocation:** `/fabrio:configure` (I'll ask for the path) or `/fabrio:configure C:\Users\you\Source`.

---

## Step 1 — Get the source root

If a path was passed as an argument, use it. Otherwise ask:

> What's the absolute path to the folder that holds your site repos? (e.g. `C:\Users\you\Source` or `/Users/you/Source`)

Use the path as given — do not invent or normalize slashes beyond trimming surrounding whitespace/quotes. It does not need to exist yet, but warn if it obviously looks wrong (relative, empty).

---

## Step 2 — Write `~/.fabrio/config.json`

The file lives in the user's home Fabrio dir — `%USERPROFILE%\.fabrio\config.json` on Windows, `$HOME/.fabrio/config.json` on macOS/Linux (the same dir as `credentials.json`). Create the `.fabrio` dir if missing, then **merge** `source_root` into the JSON, preserving any other keys already present.

- **Windows (PowerShell):**
  ```powershell
  $dir = Join-Path $env:USERPROFILE '.fabrio'; New-Item -ItemType Directory -Force $dir | Out-Null
  $f = Join-Path $dir 'config.json'
  $cfg = if (Test-Path $f) { Get-Content $f -Raw | ConvertFrom-Json } else { [pscustomobject]@{} }
  $cfg | Add-Member -NotePropertyName source_root -NotePropertyValue '<PATH>' -Force
  $cfg | ConvertTo-Json -Depth 10 | Set-Content $f -Encoding utf8
  ```
- **macOS/Linux (bash + jq, or a small node/python one-liner if jq is absent):**
  ```bash
  mkdir -p "$HOME/.fabrio"; f="$HOME/.fabrio/config.json"
  tmp=$(jq -n --arg s '<PATH>' 'input? // {} | .source_root=$s' "$f" 2>/dev/null || jq -n --arg s '<PATH>' '{source_root:$s}')
  printf '%s\n' "$tmp" > "$f"
  ```

Preserve existing keys (e.g. don't clobber the file if the user keeps other config there). Never write anything into the repo — this is home-dir config only, and it's not a secret (keys stay in `credentials.json`).

---

## Step 3 — Confirm

Read the file back and confirm:

```
✅ Source root saved to ~/.fabrio/config.json
   source_root: <PATH>

/fabrio:feature-request and /fabrio:merge-task will use it (repos resolve as <PATH>/<site relative_path>).
Re-run /fabrio:configure any time to change it. (A FABRIO_SOURCE_ROOT env var, if set, still takes precedence.)
```

---

## Step 4 — Offer the runner (optional, ask first)

Fabrio only picks work up on its own if something is watching the queue. `fabrio-runner` is
that watcher: a small Node process that polls your ready tasks and spawns one Claude Code
child per task, scoped by that task's agent profile.

**Ask before installing anything, and never enable it silently** — the plugin does not turn on
unattended work by itself, and a runner left going will open PRs without you typing a command.

> Want Fabrio to pick up ready tasks automatically? I can install `fabrio-runner` and set it to
> start with your machine. It opens PRs and prepares deliverables for review — it never merges,
> publishes, sends or spends. (You can skip this and keep running `/fabrio:run-due-jobs`
> yourself.)

If they decline, stop — everything above still works.

If they accept:

1. **Install:** `npm install -g fabrio-runner`
2. **Key:** it reads `FABRIO_API_KEY`, or `fabrio_api_key` in `~/.fabrio/credentials.json`
   (sibling to `config.json`). If neither is set, ask them to create one at **Fabrio →
   Settings → API keys** and write it to `credentials.json` — preserving existing keys, and
   never into the repo.
3. **Prove it works before scheduling it:**
   ```bash
   fabrio-runner --once --dry-run
   ```
   This prints what it would dispatch and spawns nothing. If it errors, fix that first — a
   scheduled process that fails at startup fails silently and looks like "Fabrio does nothing".
4. **Schedule it**, if they want it always on — a launchd agent on macOS, a systemd user unit
   on Linux, or a Startup shortcut on Windows. Show them the unit file rather than writing it
   without asking; it is a persistent background process and that is their call to make.
5. **Tell them what it does *not* cover:** recurring jobs and the weekly steps still come from
   `/fabrio:run-due-jobs`, so that should still be scheduled **daily** alongside it.

The runner also needs an authenticated CLI session and a user-scope tool allow-list, exactly
as unattended heartbeat runs always have — its README covers both.
