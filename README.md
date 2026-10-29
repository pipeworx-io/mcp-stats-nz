# @pipeworx/stats-nz

New Zealand official statistics (Aotearoa Data Explorer) — search Stats NZ's
911 published datasets by keyword and pull their observations, covering the
2013/2018/2023 censuses, LEED earnings, business demography, population
estimates and projections, household expenditure, corrections and justice.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `statsnz_datasets({ query, limit, offset })` — **keyless**. Keyword-search all
  911 datasets; returns dataset id, full title, description, agency, version.
- `statsnz_smoking_by_region({ region?, year?, _apiKey })` — Census smoking status (regular smoker / ex-smoker / never smoked regularly) by regional council for 2006, 2013, 2018 and 2023, with counts and percentages of people whose status was stated. Census self-report, not the Ministry of Health NZ Health Survey.
- `statsnz_observations({ dataset, key, agency, version, startPeriod, endPeriod, lastNObservations, _apiKey })` — period/value pairs with their dimension labels. A too-wide unfiltered query is auto-narrowed to totals and flagged in `narrowed` (see Traps).
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
- **An unfiltered query on a wide census table is too big to return, so the
  pack narrows it and says so.** CEN23_HAD_020 with key `all` was ~208MB of
  SDMX-JSON on 2026-10-16 (no Content-Length, no time dimension for
  `lastNObservations` to shrink), past the pack's 16MB streaming cap; that was
  24 of 44 external `statsnz_observations` calls in 48h. When the cap trips on
  an unfiltered query, the pack reads the dataset structure, pins geography
  dimensions (then the widest remaining ones, while the open cross product is
  over 250k cells) to the ROOT code Stats NZ labels "Total …" — never a
  "Total stated …" subtotal — re-queries, and returns the slice with
  `requested_key: "all"` and a `narrowed` object listing each pin and how to ask
  for the breakdown. CEN23_HAD_020 comes back as key `.9999....` (Total - New
  Zealand by regional council), 16,920 rows. A caller-supplied `key` is never
  narrowed; a too-wide one, or a dataset with no Total code to pin, fails as
  `user_error` with the dimensions and sample codes inline.
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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/statsnz_datasets \
  -H 'Content-Type: application/json' \
  -d '{"query":"household income","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/statsnz_datasets`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-stats-nz` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/stats-nz/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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
ask_pipeworx({ question: "your question about Stats Nz data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
