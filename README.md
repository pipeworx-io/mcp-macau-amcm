# @pipeworx/macau-amcm

Macau monetary data from AMCM, the Monetary Authority of Macao (Autoridade
Monetária de Macau / 澳門金融管理局): daily interbank FX mid-rates for the
Macau pataca (MOP) against ~17 currencies, and MONIA — the MOP Overnight
Index Average, Macau's overnight interbank reference rate.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `macau_fx_rates({ begin?, end?, currency? })` — dated MOP mid-rates per
  currency per Macau business day. Defaults to the last 7 days ending on the
  most recent published date; `begin`/`end` accept YYYY-MM-DD (up to ~1 year
  per call), `currency` filters by comma-separated ISO codes ("USD,HKD,CNY").
- `macau_monia({ begin?, end? })` — daily MONIA fixings in percent, same
  windowing.

## Auth

Keyless. No registration, key or cookie — the endpoints are the same JSON API
AMCM's own rate pages load their tables from.

## Data sources

- `https://www.amcm.gov.mo/api/v1.0/cms/financial_info?QueryType=ExchangeRate|Monia&Begin=YYYYMMDD&End=YYYYMMDD`
- `https://www.amcm.gov.mo/api/v1.0/cms/last_data_date?QueryType=...` — most
  recent published date, used for the default window (publication skips
  weekends and Macau public holidays).

### Traps for the next person

- **`usdMean` is not a USD rate.** Despite the name, `usdMean`/`usdMeanValue`
  is the MOP mid-rate per `unit` of each currency — the USD row itself reads
  ~8.08 (MOP per USD) and the HKD row ~1.03 (the MOP–HKD peg). Verified live
  2026-09-07. This pack exposes it as `mop_per_unit`.
- **JPY and KRW are quoted per `unit` = 100**, everything else per 1. Read the
  `unit` field before comparing across currencies.
- **`bid` changes meaning per row.** It is a second quote in each currency's
  own market convention: USD per unit for EUR/GBP/AUD/NZD, units per USD for
  JPY/CHF/SGD/SEK/NOK/etc., CNY per 100 USD, and USD/HKD on the USD row.
  Exposed as `market_quote` with that caveat — use `mop_per_unit` for
  anything MOP-denominated.
- **The FX list carries a `LIQ` row with `unit: 0`** — an AMCM liquidity
  indicator, not a currency. The FX tool filters it out.
- **An empty window is normal, not broken**: AMCM publishes only on Macau
  business days, so a weekend/holiday-only window returns zero rows. The
  tools detect that, fetch `last_data_date`, and return it in the response so
  a caller can re-ask with a real date.
- The earlier china-vertical probe (fleet #1311) assumed AMCM was
  XLS-download-only; the JSON API above is what the site's own pages use and
  needs no scraping.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "macau-amcm": {
      "url": "https://gateway.pipeworx.io/macau-amcm/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/macau-amcm/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "macau-amcm": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-macau-amcm"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-macau-amcm
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Macau Amcm data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
