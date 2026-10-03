# Hearthseed Adult MCP (Cursor plugin)

Official Hearthseed Cursor Plugin that binds the **live** Hearthseed Adult MCP into Cursor IDE. You need a Hearthseed Adult account at https://hearthseed.com.

`HouseDay = { today_tasks, cart, pantry, recipes, events }` — Adult-account scoped; the server re-reads the seat on every call.

This repo does **not** implement those tools. It points at `https://hearthseed.com/api/mcp` (streamable-http) and ships one skill that maps HouseDay fields to the live Adult MCP tool names.

| HouseDay field | Live tool |
| --- | --- |
| `today_tasks` | `today`, `tasks` |
| `cart` | `grocery` |
| `pantry` | `pantry` |
| `recipes` | `recipes` |
| `events` | `events` (`attendeeIds` = Who; update replaces the set) |

Names, actions, and keys match the live `tools/list` captured 2026-10-02. Call the live tools. Do not invent response data. Full action/key lists live in `skills/hearthseed-house-day/SKILL.md`.

Limits from that capture: 100 CART rows per `grocery` call; 100 pantry rows per `clear` / `empty`; pantry `qty` 0 drops the line; 100 ingredient lines on `recipes` `add-all` / `to_shopping`; `tasks` `check` / `strike` mark done.

## Auth (OAuth)

Cursor signs in via OAuth. **No token is needed.**

Live `mcp.json` is OAuth 2.1 PKCE via CIMD. One server only. No `CLIENT_SECRET`.

```json
{
  "mcpServers": {
    "hearthseed": {
      "url": "https://hearthseed.com/api/mcp",
      "auth": {
        "CLIENT_ID": "https://hearthseed.com/.well-known/cursor-mcp-client",
        "scopes": ["mcp"]
      }
    }
  }
}
```

This OAuth path is verified working in Cursor IDE.

Seat rules (server-enforced, not implemented here):

- One Adult per connection. Ceiling 10.
- Helper / Sitter / Child: fail-closed.
- Teen / Helper grants: grantable, default OFF. This plugin cannot grant.
- Unpaid house: server gate `house_billing_open`.

## Install locally (before the Marketplace listing)

1. Copy this repo into the Cursor local-plugin folder (copy the directory; do not symlink out of it — Cursor skips external symlink targets):

   ```bash
   mkdir -p ~/.cursor/plugins/local
   rsync -a --delete \
     --exclude .git \
     /path/to/this-repo/ \
     ~/.cursor/plugins/local/hearthseed/
   ```

2. Restart Cursor, or run **Developer: Reload Window**.
3. Open **Customize** and confirm `hearthseed` is present with the HouseDay skill and the `hearthseed` MCP server.
4. Sign in with OAuth when Cursor prompts. No Bearer token. Helper / Sitter / Child seats fail closed.

On Teams / Enterprise, local plugin imports may be gated by **Allow Local Plugin Imports** (Dashboard → Settings → Security & Identity → Marketplace and Plugins).

### Structural check (no token required)

From the repo root:

```bash
python3 - <<'PY'
import json, re
from pathlib import Path

plugin = json.loads(Path(".cursor-plugin/plugin.json").read_text())
mcp = json.loads(Path("mcp.json").read_text())
assert plugin["name"] == "hearthseed"
assert plugin["displayName"] == "Hearthseed"
assert plugin["version"] == "0.1.1"
assert plugin["author"]["name"] == "Hearthseed"
assert plugin["homepage"] == "https://hearthseed.com"
assert plugin["logo"] == "assets/logo.png"
assert "variables" not in plugin
assert Path("assets/logo.png").is_file()
used = set(re.findall(r"\$\{([A-Z0-9_]+)\}", json.dumps(mcp)))
assert used == set(), used
hs = mcp["mcpServers"]["hearthseed"]
assert hs["url"] == "https://hearthseed.com/api/mcp"
assert hs["auth"] == {
    "CLIENT_ID": "https://hearthseed.com/.well-known/cursor-mcp-client",
    "scopes": ["mcp"],
}
assert "CLIENT_SECRET" not in hs["auth"]
assert "headers" not in hs
print("shape ok")
PY
```

## What this plugin must never expose

billing, People (the People surface — not `events.attendeeIds`), delete-house, Owner, grant, Stripe, auth/kernel, calendar Connect, URL recipe scrape, `CLIENT_SECRET`.

Events write out only to calendars that are already Connected. Fan-out ceiling 100, fail-closed.

## Layout

```text
.
├── .cursor-plugin/plugin.json
├── assets/logo.png
├── mcp.json
├── skills/hearthseed-house-day/SKILL.md
└── README.md
```

No `rules/`, `hooks/`, `agents/`, or stdio server. No secrets in git.

## Advanced: non-OAuth clients

Optional and not part of Cursor IDE setup. Clients that cannot run the CIMD OAuth block may send `Authorization: Bearer <Adult token>` to `https://hearthseed.com/api/mcp` instead. Mint that token in Hearthseed (SUPER POWERS → Connect Grok Bot or Copy Token). Do not put it in `plugin.json` or commit it. Do not add a second MCP server to the live `mcp.json`.
