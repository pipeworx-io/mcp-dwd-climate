# @pipeworx/dwd-climate

Daily climate observations from Germany's national weather service, the Deutscher Wetterdienst (DWD) Climate Data Center: temperature, precipitation, sunshine, snow depth, wind, humidity, pressure and cloud cover for ~1,400 current and closed stations, with DWD quality flags, from each station's first record (some back to the 1800s) up to yesterday.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `dwd_stations(name? | lat+lon, active_only?, limit?)` — find stations by name (umlauts optional) or nearest to a coordinate; returns station id, federal state, elevation, coordinates and the first/last observation date.
- `dwd_daily(station_id, from?, to?)` — per-day observations for one station over a date range (max 1000 days per call, `next_from` when truncated), with units and quality levels.
- `dwd_latest(station_id)` — the most recent daily observation for one station, usually yesterday.

Every response carries `source`, `attribution` and `data_as_of` (the Last-Modified time of the upstream file that answered). A station number that does not exist, a name that matches nothing, or a range with no observations returns `found: false` with `data_as_of`, never an empty array. Upstream failures throw.

## Auth

Keyless.

## Data source

- Deutscher Wetterdienst (DWD) Climate Data Center — daily climate observations (KL) for German stations, in two archives: the daily-updated recent archive (roughly the last 500 days) and the quality-controlled historical archive (from each station's first record to the close of the last full year), plus DWD's station description list.

Each call reads only the one station's data it needs, live. Where a range spans both archives, the historical (quality-controlled) value wins for any day both cover; `archives_read` says which archive(s) answered.

### Reuse terms

DWD Climate Data Center terms of use (status May 2024), quoted in full:

> The Creative Commons BY 4.0 - Licence "CC BY 4.0" apply. For detailed information see https://www.dwd.de/copyright

Attribution ("Source: Deutscher Wetterdienst") is included in every response.

### Field notes

- Station ids are 5-digit zero-padded; the tools accept them with or without leading zeros.
- `-999` upstream = missing, returned as `null`. `QN_3` is the quality level for the wind columns (FX, FM), `QN_4` for everything else.
- `RSKF` (precipitation form) is a code: 0 none, 1 rain, 4 unknown form, 6 rain, 7 snow, 8 rain and snow, 9 missing.
- Many automatic stations report no sunshine duration (`SDK`); `null` there is upstream, not a parse failure.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dwd-climate": {
      "url": "https://gateway.pipeworx.io/dwd-climate/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dwd-climate/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/dwd_stations \
  -H 'Content-Type: application/json' \
  -d '{"name":"Dresden","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/dwd_stations`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-dwd-climate` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/dwd-climate/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "dwd-climate": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dwd-climate"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dwd Climate data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
