# @pipeworx/bonhams

Realized auction prices from Bonhams — search past ("sold") lots by artist or
maker and get the hammer price, the hammer price including buyer's premium
(the actual realized/sold price), estimate range, sale date and department;
pull full lot detail (description, condition, images) for one lot.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `bonhams_results_search(query, category?, page?, limit?)` — search Bonhams'
  sold-lot archive by artist/maker/keyword. `category` is a free-text filter
  matched against the department name (e.g. "jewellery", "prints", "books").
- `bonhams_lot_details(auction_id, lot_no, slug?)` — full detail for one lot,
  keyed by the `auction_id`/`lot_no` a `bonhams_results_search` result carries.

## Auth

Keyless. Public pages, no login required.

## Data sources

- `https://www.bonhams.com/search/?chronology=past&query=<q>&page=<n>` — the
  site's own past-lots search page. It's a Next.js app that server-renders the
  FULL result set into a `__NEXT_DATA__` JSON blob
  (`pageProps.lotData.auctionLots[]`) — a plain `fetch()` gets real data, no
  separate XHR/API call needed. Each entry carries `price.hammerPrice` (before
  premium) and `price.hammerPremium` (the realized price a buyer actually
  paid, matching the "Sold for $X inc. premium" text shown on the page) plus
  `estimateLow`/`estimateHigh`, `department.name`, `hammerTime.datetime`,
  `auctionId`, and `lotNo` (which is `{full, letter, number}` — `full` is
  sometimes absent, fall back to `number`).
- `https://www.bonhams.com/auction/<auctionId>/lot/<lotNo>/<slug>/` — the lot
  detail page, same SSR pattern, one `"lot":{...}` object with richer fields:
  `sDesc`/`sCatalogDesc` (HTML description), `dHammerPrice`, `dHammerPremium`,
  `dEstimateLow`/`dEstimateHigh`, `images[]`. **The slug in the URL is not
  validated** — any string works as long as `auctionId`/`lotNo` match, so a
  caller doesn't need Bonhams' exact descriptive slug.
- robots.txt (checked 2026-09-05) disallows only `/ldc/`, `/vms-assets/`,
  `*aggregate$`, `*head_image*` — none of which this pack touches. No
  crawl-delay declared for `User-agent: *`.

## Survey notes for the other two houses (Sotheby's, Phillips)

Bonhams was shipped first because its realized prices are plain SSR HTML with
no gate — the easiest shape found. Quick probe notes on the other two,
recorded for whoever builds their packs next:

- **Sotheby's** — robots.txt sets `Crawl-delay: 15` for `User-agent: *` and
  disallows `/bsp-api/*` (their actual API path) plus all PDF exports. Results
  pages are likely a genuine client-side API call behind `/bsp-api/`, which
  robots explicitly blocks — worth checking whether a *different*,
  non-disallowed endpoint serves the same data before assuming it's blocked
  outright.
- **Phillips** — robots.txt disallows `/Search`, `/search`, `/SEARCH`,
  `/bin/`, `/phillips/otis`, `/*/filter/`. The obvious keyword-search path is
  blocked case-insensitively; a lot-detail page reached some other way (e.g.
  from a sitemap or a department listing) may still be fair game — not probed
  in this session.
- **Christie's** — out of scope per the build plan (closed, internal API).

Bonhams proved the per-house-pack shape works, so follow-on fleet tasks were
filed for the other two: **#1280 (Sotheby's)** and **#1281 (Phillips)**, each
carrying the probe notes above.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bonhams": {
      "url": "https://gateway.pipeworx.io/bonhams/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bonhams/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/bonhams_results_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"Picasso"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bonhams_results_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "bonhams": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bonhams"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-bonhams
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bonhams data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
