# crs-statusline

A two-line [Claude Code](https://docs.anthropic.com/en/docs/claude-code) statusline for users of [Claude Relay Service](https://github.com/zhengyb/claude-relay-service). Shows your local session info on top, and the upstream Claude account's OAuth quota plus your daily cost below.

```
Sonnet · my-project · $0.35 · 12m12s
Upstream Usage: 5h 16% (3h29m), 7d 43% (3d), sonnet 21% (3d); My Daily Usage: $14.85/$200
```

- **Top line** — from Claude Code's stdin JSON, recomputed on every render:
  - `model.display_name` · `workspace.current_dir` (basename) · `$cost.total_cost_usd` · `cost.total_duration_ms`
  - Any field missing is silently skipped; if all four are missing, the top line is omitted.
- **Bottom line** — from the relay's `/v1/session-usage` endpoint, cached locally for 60 s:
  - **Upstream Usage**: 5h / 7d / sonnet OAuth windows with utilization % and reset countdowns (data comes from `api.anthropic.com/api/oauth/usage` via the relay)
  - **My Daily Usage**: today's rated cost vs the API Key's daily limit (`$NA` when no limit set)

## Requirements

- **Node.js 18+ and npm** must be installed globally on your system first (the script itself has zero npm deps and uses only Node built-ins, but Claude Code invokes it via `node`). Verify with `node -v && npm -v`. Install command: `sudo apt install nodejs npm`.
- Claude Code 2.1+
- A [Claude Relay Service](https://github.com/zhengyb/claude-relay-service) backend with the `/v1/session-usage` endpoint enabled (`STATUSLINE_USAGE_ENABLED=true` in its `.env`)

## Install

### Via Claude Code plugin (recommended)

From within Claude Code:

```
/plugin marketplace add zhengyb/Claude-Code-my-statusline
/plugin install crs-statusline@crs-marketplace
/reload-plugins
/crs-statusline:setup
```

`/reload-plugins` is required between install and setup so Claude Code registers the newly added slash command (otherwise `/crs-statusline:setup` will fail with `Unknown command`).

The `setup` slash command downloads the script to `~/.claude/crs-statusline.js` and patches `~/.claude/settings.json`. Restart Claude Code afterwards.

### Manual

```bash
mkdir -p ~/.claude
curl -fsSL -o ~/.claude/crs-statusline.js \
  https://raw.githubusercontent.com/zhengyb/Claude-Code-my-statusline/main/crs-statusline.js
```

Then add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "node ~/.claude/crs-statusline.js"
  }
}
```

Restart Claude Code.

## Environment

The statusline inherits the Claude Code process's environment. It needs:

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_BASE_URL` | Relay base URL with `/api` suffix (e.g. `http://your-relay:3000/api`) |
| `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` | Your `cr_` prefixed API key |

Both are normally already set when you use Claude Code against a relay; this plugin doesn't add anything new.

## Display states

| State | Output |
|-------|--------|
| All upstream + cost present | `Upstream Usage: 5h 42% (2h13m), 7d 18% (4d), sonnet 9% (4d); My Daily Usage: $1.23/$10` |
| No daily limit set on the key | `… ; My Daily Usage: $121.10/$NA` |
| Server returned stale upstream snapshot | `Upstream Usage: ~5h 90% (1m), …` (the `~` prefix) |
| API Key has no resolvable Claude account | `Upstream Usage: (暂无数据); My Daily Usage: …` |
| Account exists but isn't Claude OAuth (Setup Token / Console) | `Upstream Usage: (账号无配额数据); …` |
| Relay unreachable or env vars missing | `Claude —` |

## How it works

1. Claude Code launches `node ~/.claude/crs-statusline.js` on every statusline render and pipes a JSON blob (session id, model, workspace, cost) to its stdin.
2. The script parses stdin to build the top line.
3. It looks up a 60 s local cache (per `session_id`) under `${tmpdir}/claude-relay-statusline-*.json`. On a cache hit it prints immediately.
4. On a miss, it `GET`s `{ANTHROPIC_BASE_URL}/v1/session-usage?session={session_id}` with a 2 s timeout, formats the response into the Usage line, and writes the cache.
5. Any error is swallowed; the script always exits 0 with a sane fallback so the statusline never crashes Claude Code.

The full source is a single file — [`crs-statusline.js`](./crs-statusline.js) — read it before you install.

## License

MIT — see [LICENSE](./LICENSE).
