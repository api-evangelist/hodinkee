---
name: hodinkee-find-a-watch
description: >-
  Search the HODINKEE Shop catalog and return accurate, correctly-priced
  candidates for a buyer's request, using the store's anonymous MCP endpoint.
api: HODINKEE Shop Agent Commerce (UCP / MCP)
endpoint: https://shop.hodinkee.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-22'
method: generated
source: mcp/hodinkee-ucp-mcp-tools.json
---

# Find a watch on HODINKEE Shop

Read-only. Nothing in this skill spends money or creates state.

## Before you start

- Endpoint: `POST https://shop.hodinkee.com/api/ucp/mcp`,
  `Content-Type: application/json`, `Accept: application/json, text/event-stream`.
- No credential is required. **Every** tool call must carry
  `meta.ucp-agent.profile` — a resolvable URI for your own published agent
  profile. Omit it and the call fails with HTTP 422 and JSON-RPC error `-32001`
  (`invalid_profile_url`) before your arguments are even read.
- A narrower anonymous tool set is also available at
  `POST https://shop.hodinkee.com/api/mcp`, which does **not** require the agent
  profile. Use it when you only need `search_catalog` or `get_product_details`.

## Steps

1. **Search.** Call `search_catalog` with `catalog.query` set to the buyer's
   words. Add `catalog.filters.categories` and `catalog.filters.price.min` /
   `.max` when the buyer gave a budget. Set
   `catalog.context.address_country` and `catalog.context.currency` — pricing and
   availability depend on them.
2. **Page.** Results default to 10, maximum 250. Take
   `pagination.cursor` from the response and pass it back as
   `catalog.pagination.cursor` only when the buyer asks for more. Do not
   pre-fetch pages the buyer did not ask for.
3. **Confirm.** Call `lookup_catalog` with `catalog.ids` for several candidates
   at once, or `get_product` with `catalog.id` for one, before quoting anything.
   Search results are for ranking; the product record is for quoting.

## Quoting a price correctly

Prices come back as **integers in ISO 4217 minor units** paired with a currency
code. `{"amount": 17500, "currency": "USD"}` is **$175.00**, not $17,500.
Divide by 100 for two-decimal currencies. Zero-decimal currencies such as JPY
are already whole units. This is the single easiest way to be wrong by 100x on
this surface — convert before you speak.

## Rules

- Rate limiting is per IP and undocumented in size. Back off on `429`; there is
  no `Retry-After` or `RateLimit-*` header to read, so use exponential backoff.
- Link the buyer to the `url` on the product record rather than reconstructing a
  URL from the handle.
- Stock on limited editions moves. Re-read the product immediately before you
  hand off to a cart.
