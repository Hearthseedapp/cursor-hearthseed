---
name: hearthseed-house-day
description: >-
  Use when the user asks about a Hearthseed house day — today, tasks, grocery
  cart, pantry, recipes, or Adult-authored events. Call only the live Adult MCP
  tools today, grocery, events, pantry, recipes, and tasks at
  https://hearthseed.com/api/mcp. Never invent API data, REST paths, or tools
  this plugin does not expose.
---

# Hearthseed HouseDay

This plugin binds Cursor to the **live** Hearthseed Adult MCP at `https://hearthseed.com/api/mcp`. It does not implement tools, scrape recipes, or invent a REST surface.

Cursor signs in via OAuth. No token is required.

Tool names, actions, and keys below match the live `tools/list` captured 2026-10-02. Call those live tools. Do not invent response rows.

## Domain type

`HouseDay = { today_tasks, cart, pantry, recipes, events }`

Adult-account scoped. The server re-reads the seat on every call. Do not reuse a prior-turn seat, house, or bind from memory.

## HouseDay field → live tool

| HouseDay field | Live tool | Call when |
| --- | --- | --- |
| `today_tasks` | `today` then `tasks` | Today's board (`today`); checklist mutations (`tasks`) |
| `cart` | `grocery` | SHOPPING / CART, including move-to-pantry |
| `pantry` | `pantry` | What is in the house / stock |
| `recipes` | `recipes` | COOK BOOK. No URL scrape. |
| `events` | `events` | Adult-authored calendar items, including Who (`attendeeIds`). Write-out only to already-Connected calendars. Fan-out ceiling 100, fail-closed. |

If the live tool list does not include a name in this table, stop. Tell the user the server did not advertise it. Do not guess a REST fallback. Do not add any tool the server does not list.

## Live tools (2026-10-02 `tools/list`)

Every call is Adult-scoped. Pass `householdId` when the tool asks for it. Use only the keys listed for that tool.

### `today`

Read this Adult's Today board in the houses they act in.

Keys: `householdId`

No `action`.

### `grocery` → HouseDay `cart`

SHOPPING and CART for a house this Adult acts in. `to_pantry` moves CART like Add all to pantry. At most 100 CART rows in one call.

Actions: `list` \| `add` \| `check` \| `return` \| `remove` \| `to_pantry` \| `add-all-to-pantry`

Keys: `action`, `householdId`, `text`, `id`, `qty`

### `events` → HouseDay `events`

List, add, update, or delete events this Adult authored. `attendeeIds` is the event's Who list (array of strings). On `update` / `edit` / `patch` it **replaces** the whole set.

Actions: `list` \| `add` \| `update` \| `edit` \| `patch` \| `delete` \| `remove`

Keys: `action`, `householdId`, `id`, `title`, `startsAt`, `endsAt`, `notes`, `allDay`, `timeZone`, `attendeeIds`

### `pantry` → HouseDay `pantry`

PANTRY stock for a house this Adult acts in. `qty` 0 drops the line. At most 100 rows in one `clear` / `empty`.

Actions: `list` \| `add` \| `qty` \| `remove` \| `delete` \| `clear` \| `empty`

Keys: `action`, `householdId`, `text`, `id`, `qty`

### `recipes` → HouseDay `recipes`

COOK BOOK for a house this Adult acts in. `add-all` and `to_shopping` take at most 100 ingredient lines. No URL scrape — use the recipe the server already has.

Actions: `list` \| `get` \| `create` \| `add-all` \| `to_shopping` \| `update` \| `edit` \| `patch` \| `delete` \| `remove`

Keys: `action`, `householdId`, `id`, `name`, `ingredients`, `instructions`, `notes`

### `tasks` → HouseDay `today_tasks`

Adult Today tasks in houses this Adult acts in. `check` and `strike` mark the task done. `delete` and `remove` drop the row.

Actions: `list` \| `add` \| `check` \| `strike` \| `delete` \| `remove`

Keys: `action`, `householdId`, `id`, `title`

## Auth and seat (read, do not implement)

Cursor IDE binds with OAuth (CIMD client id `https://hearthseed.com/.well-known/cursor-mcp-client`, scope `mcp`). No Bearer token.

- One Adult per connection. Ceiling 10 connections.
- Helper / Sitter / Child: fail-closed. Do not retry as a different seat.
- Teen / Helper grants: grantable, default OFF. Do not attempt to grant.
- Unpaid house: server gate `house_billing_open`. Report that gate. Do not open billing.

GET `https://hearthseed.com/api/mcp` may return 200 as a probe. POST needs an OAuth bind. A missing bind is a bind failure, not a reason to invent data.

## Never invent API data

- Only state facts returned by a live HouseDay tool in this session (`today`, `grocery`, `events`, `pantry`, `recipes`, `tasks`).
- Empty list, error, or unpaid gate is a real answer. Do not fill from training data or a previous house.
- Do not reconstruct pantry, cart, recipes, tasks, or events from conversation if the tool did not return them.

## Do-nots

Never call, describe how to call, or reimplement:

- billing, Stripe, or `house_billing_open` as a writable surface
- People (the People surface — not the event Who list on `events.attendeeIds`)
- delete-house
- Owner
- grant (including Teen / Helper grants)
- auth / kernel
- calendar Connect (events write-out to already-Connected calendars only)
- URL recipe scrape
- any REST API you invent
- a local / stdio stand-in for these tools
- any MCP tool name other than `today`, `grocery`, `events`, `pantry`, `recipes`, `tasks`
