# @pipeworx/ga-medicaid-rates

Georgia Medicaid FQHC (federally qualified health center) and rural health clinic (RHC) per-visit PPS rates, per Medicaid provider ID and federal fiscal year, for FFY 2025, 2026 and 2027. From the Georgia Department of Community Health (DCH).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2007+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ga_clinic_rates(provider_id?, program?, ffy?, min_rate?, max_rate?, limit?)` — rate rows. Give a Medicaid provider ID to see its rate in every year held (a 9-digit prefix matches every suffix), or browse a program and year, or filter by rate range. An unmatched filter returns provider IDs that start alike and the valid values, never a bare empty.
- `ga_rate_summary(program?, ffy?, group_by?)` — which lists exist for each federal fiscal year, how many provider IDs each carries, min/median/max/mean per-visit rate, optionally grouped by program or fiscal year.

## Auth

Keyless.

## Data sources

- Georgia Department of Community Health (DCH), Medicaid FQHC and rural health clinic PPS rate lists: four lists per federal fiscal year (FQHC, freestanding RHC, hospital-based RHC, critical-access RHC). Public, no account.

Things worth knowing:

- **The lists carry no clinic names.** Each row is a Medicaid provider ID and a rate: no name, address, NPI or county. This pack says so in every response and does not guess names. DCH's provider directory is an interactive search with no downloadable ID-to-name file, so no join is made. Look up by `provider_id`.
- **Organisations only.** The pack drops rows whose provider name reads as a person; with no names in the lists there is nothing to judge, so `individual_rows_dropped` is 0.
- Rates are per provider ID, per visit (PPS), not per procedure code. Federal fiscal year 2027 runs 2026-10-01 to 2027-09-30. Fee-for-service only: managed-care plans may pay under their own contracts.
- Provider IDs are strings; leading zeros matter.
- Publisher repeats and contradictions are handled in the open. An ID printed twice with the same rate is counted once (`repeated_ids_dropped` in `ga_rate_summary`). An ID printed twice with different rates inside one list (FFY 2025 critical-access list: `003303780A`, 94.00 and 93.99) is left out and named in `ids_listed_with_two_rates_and_left_out`.
- Every response carries `data_as_of` (the date DCH posted the newest lists), `captured_at` and the source.
- This is a supply-side build with no measured demand yet; it exists because a state-by-state Medicaid rate set was asked for.

## Stored data and refresh

DCH posts the lists as PDFs with no query interface, so `scripts/bake-index.mjs` reads them into `src/ga-rates-data.ts` (about 1,400 rows, 100 KB), registered in `workers/gateway/src/pack-baked-indexes.json`; the deploy uploads it to KV and the gateway injects it on the first call into this pack, so nothing is evaluated at module scope. Refresh with `node mcps/ga-medicaid-rates/scripts/bake-index.mjs` (needs poppler's `pdftotext` on PATH). DCH posts one new set each year, effective 1 October; when it appears, move the previous newest set into the `PRIOR` list in the script. The script refuses a changed header, an unknown title or line, a period that does not match the fiscal year, a row count below its floor, more than three contradictory IDs in a list, or a non-positive rate.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ga-medicaid-rates": {
      "url": "https://gateway.pipeworx.io/ga-medicaid-rates/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ga-medicaid-rates/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2007+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ga_clinic_rates \
  -H 'Content-Type: application/json' \
  -d '{"provider_id":"003140498A"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ga_clinic_rates`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-ga-medicaid-rates` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/ga-medicaid-rates/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "ga-medicaid-rates": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ga-medicaid-rates"]
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
ask_pipeworx({ question: "your question about Ga Medicaid Rates data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
