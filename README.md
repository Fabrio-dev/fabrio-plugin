# Fabrio Plugin (Claude Code + Codex)

Adds Fabrio's workflow commands to Claude Code and native skills to Codex:

- `/fabrio:feature-request`, `/fabrio:ops-heartbeat`, `/fabrio:merge-task`, `/fabrio:generate-plan`, `/fabrio:revise-plan`, `/fabrio:consolidate-learnings`

The full set also includes `configure`, `execute-task`, `feature-chain`, `plan-job`, and
`run-generator`. In Codex, invoke the same workflows as `$fabrio:<name>` or in natural language.

The commands read and write your Fabrio data through the **Fabrio MCP server**, which you connect once with a per-account API key. Installing the plugin gives you the commands; connecting the MCP gives them their data. Two quick steps:

## 1. Install the commands

```
claude plugin marketplace add Fabrio-dev/fabrio-plugin
claude plugin install fabrio@fabrio-dev
```
(Or, in an interactive Claude Code session: `/plugin marketplace add Fabrio-dev/fabrio-plugin` then `/plugin install fabrio@fabrio-dev`.)

## 2. Connect your account's data (MCP)

In Fabrio, go to **Settings → API keys → Create key**, then copy the **Connect command** shown there and run it — it's a one-liner with your key already inlined:

```
claude mcp add --transport http -s user fabrio https://fabrio.dev/api/mcp --header "Authorization: Bearer fab_live_YOUR_KEY"
```

(`-s user` registers it at user scope, so it works from **every** folder you open Claude
Code from — not just the one you ran this in.) Restart Claude Code, then verify:

```
claude mcp list        # → fabrio: connected
```

The server is named `fabrio`, so its tools appear as `mcp__fabrio__*` — which is what the commands expect.

## Codex installation

Install the same public marketplace and plugin:

```bash
codex plugin marketplace add Fabrio-dev/fabrio-plugin
codex plugin add fabrio@fabrio-dev
```

Keep the Fabrio key in the environment used to launch Codex, then register the MCP server:

```bash
export FABRIO_API_KEY="fab_live_YOUR_KEY"
codex mcp add fabrio --url https://fabrio.dev/api/mcp \
  --bearer-token-env-var FABRIO_API_KEY
```

Restart Codex, open a new task, and run `$fabrio:configure`. For recurring operations, create a
Codex automation whose prompt is `Run $fabrio:ops-heartbeat`; plugin installation never starts
unattended work by itself.

## Switching accounts

Each account has its own API key. To point Claude at a different account, re-run the connect command with that account's key (or use the repo's `scripts/use-account` helper if you have Fabrio checked out). One account is active at a time.

## Prerequisites

- A Fabrio account + an API key (Settings → API keys; starts with `fab_live_`).
- `gh` (GitHub CLI) authenticated and your site repos checked out — the workflow commands run `git`/`gh`/`npm` in those repos.

### Tell the commands where your repos live (`FABRIO_SOURCE_ROOT`)

The commands build each repo's path as `{FABRIO_SOURCE_ROOT}/{site.relative_path}`, so
they need to know the local folder that holds your site checkouts. Set it in **one** of
these ways — Claude Code reads it from its own environment, and it does **not** read
Fabrio's `.env.local` (that file only configures the app itself):

- **Claude Code `settings.json` (recommended — works from any folder).** Add to
  `~/.claude/settings.json`:
  ```json
  { "env": { "FABRIO_SOURCE_ROOT": "C:\\Users\\you\\Source" } }
  ```
  (macOS/Linux: `"/Users/you/Source"`.) This injects the variable into every Claude Code
  session, so the commands find your repos no matter which directory you run them from.

- **A real OS/shell environment variable** — a Windows user env var, or
  `export FABRIO_SOURCE_ROOT=/Users/you/Source` in your shell profile — set **before**
  you launch `claude`.

If it's unset, the command just asks you for the path once per run — nothing breaks, it's
only less hands-off.

## Security

- Your API key lives only in your local MCP config; it's never committed here.
- A key is scoped to exactly one Fabrio account and can't touch any other account's data.
- Revoke a key anytime in Fabrio → Settings → API keys; revocation is immediate.

---

*The Claude commands and Codex skills here are generated from the Fabrio app repo (`.claude/commands/`) via its `scripts/sync-plugin.mjs`. Edit workflows there, not here.*
