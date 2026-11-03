# @pipeworx/noirlab-datalab

NSF NOIRLab's Astro Data Lab — the DECam, Mayall and Blanco survey catalogues (Legacy
Surveys/DECaLS, DES, DELVE, DECaPS, SMASH, NSC) alongside mirrored all-sky catalogues (Gaia DR3,
AllWISE, CatWISE, 2MASS, SDSS, DESI), queryable as database tables. Around 130 schemas and tens of
billions of rows.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1867+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `datalab_tables(schema?, table?, limit?)` — list the survey data releases, one release's tables,
  or one table's columns. Call it first: table names are survey-specific and a guessed name is a
  query error, not an empty result.
- `datalab_query(sql, limit?)` — run a read-only ADQL `SELECT`. A `TOP n` is injected when you omit
  one, so a bare query cannot scan a billion-row table unbounded.
- `datalab_cone_search(table, ra, dec, radius?, columns?, limit?)` — sources within a radius of a
  sky position, nearest first, with exact angular separation in degrees.

## Auth

Keyless — **but only through the IVOA TAP endpoint.** See the first trap below.

## Data sources

- <https://datalab.noirlab.edu/tap/sync> — `REQUEST=doQuery&LANG=ADQL&FORMAT=csv&QUERY=…`.
  Anonymous, and the only door this pack uses.

## Traps

- **`/query/query?sql=` is NOT anonymous, despite being the documented front door.** It is what the
  `datalab` Python client uses and what the docs lead with, but without a login it answers
  `401 The provided security token is invalid` — and the widely-quoted
  `X-DL-AuthToken: anonymous.0.0.anon_access` answers **HTTP 200** with the body
  `Error in query(). Try checking that you are logged in.` A 200 carrying a failure is the worst
  shape a failure can take, and it is why this pack speaks TAP only. Measured 2026-09-17.
- **TAP reports every error as HTTP 200.** Syntax errors, unknown tables, timeouts: all come back
  200 with a VOTable whose single `INFO` element carries `QUERY_STATUS="ERROR"`. Read as data
  that is an empty result set. Every response is checked for that element before it is parsed.
- **ADQL geometry is not implemented, and neither is Q3C.**
  `CONTAINS(POINT('ICRS',ra,dec), CIRCLE('ICRS',…))` parses and is handed to PostgreSQL verbatim,
  which answers `function circle(numeric, numeric, numeric) does not exist`. The Q3C functions the
  backend actually has (`q3c_radial_query`) are rejected by the ADQL parser one layer up. So both
  obvious cone-search spellings fail. `datalab_cone_search` builds the search from trigonometry
  both layers accept: an indexable RA/Dec box (RA widened by `1/cos(dec)`, wrapping at 0h) to
  narrow the scan, then an exact great-circle distance to trim the box corners off and order by
  true separation.
- **Row limits are `SELECT TOP n`, not `LIMIT n`.** ADQL, not PostgreSQL.
- **Zero rows is usually footprint, not absence.** The DECam surveys cover the southern sky and the
  Galactic plane selectively, so a northern target legitimately returns nothing from `ls_dr10` or
  `des_dr2`. The zero-result note says to re-check against `gaia_dr3.gaia_source`, which is all-sky.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "noirlab-datalab": {
      "url": "https://gateway.pipeworx.io/noirlab-datalab/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/noirlab-datalab/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1867+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/datalab_query \
  -H 'Content-Type: application/json' \
  -d '{"sql":"SELECT ra, dec, phot_g_mean_mag FROM gaia_dr3.gaia_source WHERE phot_g_mean_mag < 6","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/datalab_query`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-noirlab-datalab` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/noirlab-datalab/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "noirlab-datalab": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-noirlab-datalab"]
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
ask_pipeworx({ question: "your question about Noirlab Datalab data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
