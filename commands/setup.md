---
description: Install or update crs-statusline (Claude Relay Service statusline)
allowed-tools: Bash, Read, Write, Edit
---

# crs-statusline Setup

You are installing or updating crs-statusline, a two-line statusline
for Claude Code users of [Claude Relay Service](https://github.com/zhengyb/claude-relay-service).
This skill is idempotent: safe to run for both first install and subsequent updates.

Follow these steps in order. If any step fails, stop and explain the issue to the user.

## Step 1: Check prerequisites

Run: `command -v node`

If Node is not found, tell the user to install Node.js 18 or newer and stop.

## Step 2: Download the latest script

Always fetch from the main branch to get the latest version:

```bash
mkdir -p ~/.claude
curl -fsSL -o ~/.claude/claude-statusline.js \
  https://raw.githubusercontent.com/zhengyb/Claude-Code-my-statusline/main/statusline.js
```

## Step 3: Configure statusline

Read `~/.claude/settings.json` with the Read tool. Then use the Edit tool to add
or update the `statusLine` key:

```json
"statusLine": {
  "type": "command",
  "command": "node ~/.claude/claude-statusline.js"
}
```

If `statusLine` already exists, update the `command` value. If it does not
exist, add it as a top-level key.

## Step 4: Check environment (informational)

Tell the user this statusline reads two environment variables from the
Claude Code process:

- `ANTHROPIC_BASE_URL` — your relay base URL (must end with `/api`, e.g. `http://your-relay:3000/api`)
- `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` — your `cr_` prefixed API key

These are normally already exported in any shell that uses Claude Code with a
relay. If either is missing, the statusline degrades to `Claude —`.

The relay must also have the `/v1/session-usage` endpoint enabled:
set `STATUSLINE_USAGE_ENABLED=true` in the relay's `.env` and restart it.

## Step 5: Confirm

Tell the user:

- crs-statusline has been installed (or updated) successfully.
- Restart Claude Code (or start a new session) to see the statusline.
- To update later: run `/crs-statusline:setup` again.
- To remove: delete the `statusLine` block from `~/.claude/settings.json` and
  `~/.claude/claude-statusline.js`.
