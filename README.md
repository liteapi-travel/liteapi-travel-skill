# LiteAPI travel skill

Agent Skill that teaches Cursor, Claude Code, and other coding agents how to build a travel booking experience on the [LiteAPI](https://docs.liteapi.travel) (Nuitee Connect) REST API.

The agent writes **your** app: server-side routes for hotel search, rates, prebook, the Payment SDK, and book. It does not put the API key in the browser. Flights, experiences, eSIMs, loyalty, and vouchers are mapped in `references/capabilities.md`. The hotel flow matches the [vibe-coding prompt](https://docs.liteapi.travel/reference/prompt-for-vibe-coding-tools).

This repo is the skill. The folder that contains `SKILL.md` is what the coding tool loads.

## Install in Cursor

From the project you want the agent to work in:

```bash
mkdir -p .cursor/skills
git clone https://github.com/liteapi-travel/liteapi-travel-skill.git .cursor/skills/liteapi-travel
```

Reload the window. Ask for a booking site, a hotel search, or a whitelabel checkout. The skill loads when the request is about LiteAPI, Nuitee Connect, rates, prebook, or a travel booking flow.

To use it in every project on your machine, clone the same folder into your user skills directory instead of `.cursor/skills/` (Cursor: `~/.cursor/skills/liteapi-travel`).

## Install in Claude Code

```bash
mkdir -p .claude/skills
git clone https://github.com/liteapi-travel/liteapi-travel-skill.git .claude/skills/liteapi-travel
```

User-wide: clone into `~/.claude/skills/liteapi-travel`.

## API key

Create a key in the Nuitee Connect dashboard. Sandbox keys start with `sand_` or `sandbox_`. Production keys start with `prod_`.

Set it where the **server** reads it, not in client code:

```bash
export LITEAPI_API_KEY=sand_your_key_here
```

Do not commit the key. `.gitignore` ignores `.env`.

For local sandbox payments, the skill tells the agent to show the test card `4242424242424242`.

## What the agent will build

For a hotel whitelabel, unless you ask for a different product:

1. Destination autocomplete, or a free-text "search by vibe".
2. Hotel list with a live price.
3. Hotel page with rates grouped by room.
4. Guest details, then the LiteAPI Payment SDK.
5. Booking confirmation with `bookingId` and `hotelConfirmationCode`.

All LiteAPI calls go through server routes you own.

## MCP

LiteAPI also ships an MCP server for agents that call tools directly: [MCP server docs](https://docs.liteapi.travel/reference/mcp-server).

```
https://mcp.liteapi.travel/api/mcp?apiKey=YOUR_API_KEY
```

Use MCP to explore the account. Use this skill when the agent should generate an app that calls REST.

## Layout

```
SKILL.md                    # instructions the agent loads
references/hotels.md        # hotel request and response shapes
references/capabilities.md  # flights, experiences, eSIM, loyalty, vouchers, static data
references/security.md      # server-side key handling, sandbox vs live, payments
```

## Docs

- [Prompt for vibe coding tools](https://docs.liteapi.travel/reference/prompt-for-vibe-coding-tools)
- [Endpoints overview](https://docs.liteapi.travel/reference/api-endpoints-overview)
- [MCP server](https://docs.liteapi.travel/reference/mcp-server)
