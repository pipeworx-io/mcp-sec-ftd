# @pipeworx/sec-ftd

SEC fails-to-deliver (CNS) data. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `ftd_security(symbol, periods?)` — fails history for one ticker across recent half-month files.
- `ftd_largest(period?, limit?)` — the biggest fails in one reporting period.
- `ftd_periods()` — which periods SEC has published (currently 408, back to 2009).

## The semantics are the point of this pack

**A fail-to-deliver is a settlement failure** — shares not delivered by the settlement date. The SEC
says, where it publishes this file, that fails are **not evidence of naked short selling**. Ordinary
causes include processing delays, order-entry mistakes, illiquidity in thinly traded securities, and
long sales where the seller's shares did not arrive in time.

There is a loud audience that wants the opposite inference. So the framing is not a footnote: it is
in every tool description **and** rides on every response as `what_this_is` and `not_evidence_of`.
An agent that reads only the payload still cannot honestly draw the naked-shorting conclusion.

Two more reading traps, both surfaced in the responses themselves:

- **The published number is cumulative, not daily-new.** It is total shares failing on that
  settlement date across all participants. Summing days over-counts a fail that persists, which is
  why `ftd_security` returns `peak_shares_failed` alongside the total and says so in
  `reading_note`, and why `ftd_largest` ranks by each security's worst single day rather than a sum.
- **Absence is not a clean bill of health.** SEC lists a security on a date only when its fails
  exceed the reporting threshold, so "no rows" means no *reportable* fails — not that nothing
  failed and not that the ticker is unknown. The `found: false` path says exactly that.

## Data source and cadence

`https://www.sec.gov/data-research/sec-markets-data/fails-deliver-data` — one pipe-delimited file
per **half month** (`a` = 1st–15th, `b` = 16th–month end), zipped. Measured 2026-08-25: the current
file (`cnsfails202607b.zip`) is 1.65 MB zipped, 4.6 MB raw, **73,340 rows**, covering **13,500
securities**. SEC posts a period a few weeks after it closes.

Each call reads the periods the question needs straight from SEC, so currency is simply whatever
SEC has published — `ftd_periods` reports it. A request spanning many periods costs one fetch per
half month, which is why `periods` is capped at 6.

## Two upstream traps worth knowing

- **The path is not constructible.** Recent files live under
  `/files/data/fails-deliver-data/`, older ones under `/files/data/other/fails-deliver-data/` —
  verified 2026-08-25, where `cnsfails202605b.zip` 404s on the first prefix and 200s on the second.
  The pack reads SEC's index page rather than guessing.
- **These ZIPs set the data-descriptor flag (0x08)**, so the local file header carries zeroes for
  both sizes. A reader that trusts the local header extracts nothing. Sizes are taken from the
  central directory, which always has the real values.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sec-ftd": {
      "url": "https://gateway.pipeworx.io/sec-ftd/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sec-ftd/mcp` returns the tools in the table
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

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ftd_security \
  -H 'Content-Type: application/json' \
  -d '{"symbol":"GME","periods":2}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ftd_security`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "sec-ftd": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sec-ftd"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-sec-ftd
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Sec Ftd data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
