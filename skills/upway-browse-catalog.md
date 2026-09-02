---
name: upway-browse-catalog
description: >-
  Read Upway's certified pre-owned e-bike inventory without transacting - price and
  availability research over the UCP MCP catalog tools, or over the anonymous storefront
  JSON endpoints Upway documents for read-only agents.
api: Upway Commerce (UCP MCP) API
endpoint: https://upway.co/api/ucp/mcp
transport: mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-09-02'
method: generated
source: mcp/upway-mcp-tools.json
---

# Browse the Upway catalog (read-only)

For agents that only need to read inventory — price comparison, availability watching,
research — with no cart and no purchase. Tool names and parameters come from the live
`tools/list` saved at `mcp/upway-mcp-tools.json`; the HTTP paths come verbatim from
`https://upway.co/agents.md`.

## Two ways in

**MCP (structured, recommended).** `POST https://upway.co/api/ucp/mcp`, JSON-RPC 2.0,
anonymous. Every call requires
`{"meta": {"ucp-agent": {"profile": "<your agent profile URI>"}}}`.

**Plain HTTP JSON (no MCP client needed).** Documented by Upway for read-only agents:

- `GET /collections/all` — all products
- `GET /products/{handle}` and `GET /products/{handle}.json`
- `GET /collections/{handle}` and `GET /collections/{handle}/products.json`
- `GET /search?q={query}&type=product`
- `GET /sitemap.xml`

Note that `robots.txt` on upway.co disallows `/search` and `/collections/*sort_by*` for
crawlers. Use the MCP `search_catalog` tool for search rather than the HTML search path.

## Steps

1. **Search.** `search_catalog` with `catalog.query`. Narrow with
   `catalog.filters.categories[]` (OR logic), `catalog.filters.price.min` /
   `catalog.filters.price.max` — **integers in minor currency units**, so $2,000 is
   `200000` — and `catalog.filters.available`, which defaults to `true` and returns
   only sale-ready stock.

2. **Localize.** Set `catalog.context.address_country` (ISO 3166-1 alpha-2),
   `catalog.context.currency` (ISO 4217) and `catalog.context.language` (BCP 47).
   Pricing and availability depend on it. Upway runs separate storefronts per market —
   upway.co (US), upway.fr, upway.de, upway.be, upway.nl, upway.es, upway.it — each with
   its own UCP profile and its own MCP endpoint under that domain; for a non-US buyer,
   use that market's host rather than passing a country hint to the US store.

3. **Page.** `catalog.pagination.limit` and `catalog.pagination.cursor`. The response
   field carrying the next cursor is not documented by Upway — read it off the first
   response.

4. **Resolve many at once.** `lookup_catalog` takes identifiers and returns multiple
   products or variants in one call. Prefer it over a loop of `get_product` when you
   already hold ids.

5. **Get full detail on one.** `get_product` with `catalog.id`, plus
   `catalog.selected[]` (`{name, label}` pairs) to pin a variant and
   `catalog.preferences[]` for buyer preferences. This is the only tool that returns a
   complete single product record.

## Reading the results

- Prices are `{"amount": <integer>, "currency": "<ISO 4217>"}` in minor units. Convert
  before displaying. Zero-decimal currencies such as JPY are already whole units.
- Inventory is one-of-a-kind. Upway sells individual refurbished bikes, so a result set
  goes stale quickly — re-check with `get_product` immediately before acting on it.

## Limits and etiquette

The MCP endpoint is rate limited per IP and Upway asks agents to back off on `429`. No
`RateLimit-*` headers and no `Retry-After` are returned, so use exponential backoff.
Do not call any cart or checkout tool from a read-only flow.
