# @pipeworx/stats-nz

New Zealand official statistics (Aotearoa Data Explorer) — search Stats NZ's
911 published datasets by keyword and pull their observations, covering the
2013/2018/2023 censuses, LEED earnings, business demography, population
estimates and projections, household expenditure, corrections and justice.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1715+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `statsnz_datasets({ query, limit, offset })` — **keyless**. Keyword-search all
  911 datasets; returns dataset id, full title, description, agency, version.
- `statsnz_observations({ dataset, key, agency, version, startPeriod, endPeriod, lastNObservations, _apiKey })` — period/value pairs with their dimension labels.
- `statsnz_resources({ dataset, agency, version, includeCodes, maxCodesPerList, _apiKey })` — dimensions in key order, time dimension, measure, and the code lists that turn a key like `1.2.Q` into something readable.

## Auth

Platform key `PLATFORM_STATSNZ_KEY` (in the encrypted keystore since
2026-09-18) with BYO override via `?_apiKey=`; sent upstream as an
`Ocp-Apim-Subscription-Key` header. Callers need no key of their own. Free
subscription at <https://portal.apis.stats.govt.nz/> — sign up, then subscribe
to the Aotearoa Data Explorer API product.

`statsnz_datasets` additionally works with no key at all.

## Data sources

- <https://api.data.stats.govt.nz/rest/dataflow/all/all/latest> — dataset catalogue (keyless).
- <https://api.data.stats.govt.nz/rest/data/{agency},{dataset},{version}/{key}> — observations (keyed).
- <https://api.data.stats.govt.nz/rest/datastructure/{agency}/{dataset}/{version}?references=all> — structure + code lists (keyed).

## Traps

- **The catalogue is SDMX-ML now, whatever you ask for.**
  `/rest/dataflow/all/all/latest` served SDMX-JSON until 2026-09-18 and now
  serves `application/vnd.sdmx.structure+xml; version=2.1` on *every* Accept
  header tried — `structure+json;version=1.0`, bare `application/json`,
  `structure+json;version=2.0` — keyed and keyless alike. `res.json()` on it
  throws `Unexpected token '<', "<?xml vers"`, which classifies as
  `upstream_down` and reads like an outage rather than a format change. The
  pack parses the XML. This is endpoint-specific: `/rest/datastructure/…` still
  answers structure+JSON, so don't convert that one too.
- **The flowRef commas must stay literal — encoding them is a silent zero.**
  The data path is `/rest/data/{agency},{dataset},{version}/{key}`. Their Azure
  APIM does not percent-decode path segments, and an unmatched route falls
  through to a handler that answers **401 "missing subscription key"** without a
  key and an **empty-but-valid 200** with one. Measured 2026-09-18:
  `/rest/data/STATSNZ,AGR_AGR_001,1.0/all?lastNObservations=1` → 200 with 80
  observations; the same URL with `%2C` → 401. `encodeURIComponent` on that
  segment is what made `statsnz_observations` return `count: 0` against a
  perfectly healthy dataset. `/rest/dataflow/%61ll/all/latest` → 401 too, so it
  is their routing, not this one path.
- **A zero from `statsnz_observations` is always explained, and a bad id is an
  error.** Because an unknown dataset, a malformed path and a genuinely empty
  query are indistinguishable by status on this gateway, the tool validates the
  dataset id against the keyless catalogue *before* calling the data endpoint
  (cached 30 min per isolate). An unknown id raises a 404-shaped error naming
  near matches; an id that exists but returns nothing comes back with
  `empty_reason` saying which case it is — no structure at all (routed but
  uninterpretable), a dimension `key` that filtered everything out, a period
  window with nothing in it, or a genuine upstream gap.
- **401 does not mean "bad credential" here.** Because an unmatched route also
  answers 401, a malformed path and a missing key are indistinguishable by
  status. The pack's 401 message says both possibilities out loud.
- **Observations come back FLAT, not series-keyed.** `dataSets[0].observations`
  is a map of `"0:3" → [value, …]` and `dataSets[0].series` is absent. Reading
  only `series` (as this pack first did) yields zero observations from a good
  200. Both layouts are mapped, plus dataset-level dimensions.
- **`lastNObservations` drops the collapsed dimension from the observation
  key, and it is NOT always the time axis.** With `lastNObservations=1` on
  AGR_AGR_001 the key is 2 parts for 3 declared dimensions — the year is simply
  not in the response, so rows cannot say which year they are. On INC_INC_001
  the collapsed dimension is `MEASURE_INC_INC_001`, a *measure* axis, across
  35,610 rows. The pack reports `collapsed_dimensions` plus a note rather than
  returning rows that look fully labelled.
- **`startPeriod`/`endPeriod` only bite on a declared time dimension, and Stats
  NZ mostly declares none.** `statsnz_resources` reports `time_dimension: []`
  for AGR_AGR_001, and the `role` field in the data message just echoes each
  dimension's own id, so there is no usable time marker: a dimension named
  `YEAR_*` is not a time dimension to this server. The pack therefore tests for
  the SDMX standard `TIME_PERIOD` id and, for anything else, tells the caller to
  pin the dimension through the `key` argument — naming its segment position and
  handing back a worked key such as `..<YEAR_AGR_AGR_001>`. The note used to
  recommend `startPeriod`/`endPeriod` unconditionally, which is advice that
  cannot work on a measure axis (fleet #2244).
- **`Accept: application/vnd.sdmx.data+json;version=1.0.0` is a 406.** Their
  server lists genericdata+xml 2.1 and data+json 2.0 as acceptable. Plain
  `application/json` returns SDMX-JSON 1.0 and is what this pack asks for.
- **Wrong host looks like an outage.** The older documented host
  `api.stats.govt.nz` answers **HTTP 502** from its Azure Application Gateway on
  every path including the root. A 502 reads as "their service is down" when in
  fact the service moved. The live host is `api.data.stats.govt.nz`. Do not
  "fix" this pack back to the old hostname.
- **A bare SDMX resource path 404s.** `/rest/dataflow` is `404 Resource not
  found`; only the fully-qualified
  `/rest/dataflow/{agency}/{resource}/{version}` form answers.
- Only 341 of the 911 dataflows carry a `description`; the rest are title-only,
  so keyword filtering leans on long formal titles ("Earnings for people in paid
  employment", never "wages").
- **CPI, GDP and merchandise trade are NOT in this catalogue.** Measured
  2026-09-17: 0 of 911 dataflows mention "price", "consumer" or "trade"; 705 are
  census tables. Describing the pack as covering CPI would misroute every
  inflation question to a source that cannot answer it.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "stats-nz": {
      "url": "https://gateway.pipeworx.io/stats-nz/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/stats-nz/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1715+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/statsnz_datasets \
  -H 'Content-Type: application/json' \
  -d '{"query":"household income","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/statsnz_datasets`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "stats-nz": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-stats-nz"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-stats-nz
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Stats Nz data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
