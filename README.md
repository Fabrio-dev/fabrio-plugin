# Fabrio Plugin (Claude Code)

Install this plugin to run Fabrio's AI workflows from Claude Code. It bundles:

- The **workflow slash commands** — `/fabrio:feature-request`, `/fabrio:ops-heartbeat`, `/fabrio:merge-task`, `/fabrio:generate-plan`, `/fabrio:revise-plan`, `/fabrio:consolidate-learnings`.
- The **Fabrio MCP connection** — so those commands can read and write your Fabrio account's data. You supply your own API key on install; it's stored securely and sent as a bearer token.

You do **not** need a checkout of the Fabrio app. This is all you install.

## Prerequisites

- Claude Code **v2.1.207+** (required for per-user config values to reach the MCP header).
- A Fabrio account and an API key: **Fabrio → Settings → API keys → Create key** (starts with `fab_live_`, shown once).
- `gh` (GitHub CLI) authenticated, and your site repos checked out locally — the workflow skills run `git`/`gh`/`npm` in those repos. Set `FABRIO_SOURCE_ROOT` to the folder that contains them.

## Install

```
/plugin marketplace add Fabrio-dev/fabrio-plugin
/plugin install fabrio@fabrio-dev
```

On install you'll be prompted for:

- **Fabrio API Key** — paste your `fab_live_…` key.
- **Fabrio MCP URL** — press enter to accept `https://fabrio.dev/api/mcp` (only change it for self-hosting or local dev, e.g. `http://localhost:3000/api/mcp`).

Verify:

```
/plugin        # fabrio shows as enabled
claude mcp list   # the bundled fabrio server shows connected
```

Then run a command, e.g. `/fabrio:feature-request 42` or just `/fabrio:feature-request`.

## Switching accounts

Each account has its own API key. To point Claude at a different account, update the plugin's API Key in `/plugin` config (or reinstall with the other key). One account is active at a time.

## Security

- Your API key is stored by Claude Code (Keychain / credentials file), never committed here.
- A key is scoped to exactly one Fabrio account and can't read or write any other account's data.
- Revoke a key anytime in Fabrio → Settings → API keys; revocation takes effect immediately.

---

*The command files here are generated from the Fabrio app repo (`.claude/commands/`) via its `scripts/sync-plugin.mjs`. Edit skills there, not here.*
