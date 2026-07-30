---
name: configure
description: "One-time per-machine setup — saves the source root (folder holding your site repos) to ~/.fabrio/config.json so the workflow skills can find your code."
---

# Configure

## Codex execution contract

- Invoke Fabrio workflows by skill name (for example, `$fabrio:execute-task 42`). Natural-language requests that clearly name the workflow are equivalent.
- Never invoke the Claude CLI. When this workflow calls for a headless child or delegated Fabrio workflow, delegate the named `$fabrio:*` skill to a Codex sub-agent with the same arguments, working directory, safety gates, and requested model tier when available; wait for it and inspect Fabrio state afterward. If agent delegation is unavailable, run the referenced skill inline.
- For unattended or recurring operation, use a Codex automation whose prompt invokes `$fabrio:ops-heartbeat`. A plugin install does not create or enable an automation automatically.
- Fabrio MCP access is configured separately. If it is missing, direct the user to set `FABRIO_API_KEY`, run `codex mcp add fabrio --url https://fabrio.dev/api/mcp --bearer-token-env-var FABRIO_API_KEY`, then restart Codex and open a new task.
- Preserve every workflow safety boundary below: open PRs but never merge without the explicit merge workflow, prepare external actions but never perform them, and use durable Fabrio questions/receipts instead of prompting from delegated or automated work.

One-time, per-machine setup. The workflow skills that touch code (`$fabrio:feature-request`, `$fabrio:merge-task`) build each repo's path as `{source_root}/{site.relative_path}`. This command saves your **source root** — the absolute path to the folder that holds your site repos — to `~/.fabrio/config.json` so those skills find your code without any env var.

**Invocation:** `$fabrio:configure` (I'll ask for the path) or `$fabrio:configure C:\Users\you\Source`.

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

$fabrio:feature-request and $fabrio:merge-task will use it (repos resolve as <PATH>/<site relative_path>).
Re-run $fabrio:configure any time to change it. (A FABRIO_SOURCE_ROOT env var, if set, still takes precedence.)
```
