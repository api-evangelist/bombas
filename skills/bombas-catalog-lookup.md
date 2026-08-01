---
name: Read the Bombas catalog without transacting
description: Search, look up and detail Bombas products for research, price checking
  or comparison, using either the read-only Shopify storefront JSON endpoints or the
  read methods of the store's UCP Shopping MCP endpoint.
api: mcp/bombas-ucp-shopping-2026-04-08.openrpc.json
endpoint: https://shop.bombas.com/api/ucp/mcp
operations:
- search_catalog
- lookup_catalog
- get_product
generated: '2026-07-31'
method: generated
source: https://shop.bombas.com/agents.md
---

# Read the Bombas catalog

Two independent read paths exist. Pick the cheaper one for the job.

## Path A — no authentication, no protocol (recommended for pure reading)

Bombas documents these storefront endpoints in its own `agents.md`. All are plain
`GET` on `https://shop.bombas.com`, no credentials, no agent profile:

| Purpose | Request |
|---|---|
| Browse everything | `GET /collections/all` |
| Product page | `GET /products/{handle}` |
| Product JSON | `GET /products/{handle}.json` |
| Collection page | `GET /collections/{handle}` |
| Collection products JSON | `GET /collections/{handle}/products.json` |
| Search | `GET /search?q={query}&type=product` |
| Sitemap | `GET /sitemap.xml` |

These are rate limited per IP and return HTTP 429 with body `local_rate_limited` when
you push too hard. Back off; do not retry tightly.

Do **not** read from `https://bombas.com` — the primary brand host sits behind bot
mitigation and answers 429 with an HTML challenge on every path, including
`/robots.txt`. `shop.bombas.com` is the machine-readable surface.

## Path B — UCP Shopping MCP read methods

Use these when you are already in a UCP session, or when you need structured results
consistent with the cart/checkout entities.

1. Send `meta.ucp-agent.profile` (your platform's UCP profile URI) on every call —
   without it the endpoint rejects the request with `-32001` / `invalid_profile_url`.
2. `search_catalog` — query text, filters, pagination.
3. `lookup_catalog` — batch resolve products or variants by identifier.
4. `get_product` — full product detail by identifier, with optional interactive option
   selection.

None of the three requires an idempotency key. Results carry the same UCP envelope as
the transactional methods, so check `messages[]` for `not_found`, `item_unavailable`
or `out_of_stock` before presenting anything to the buyer.

## What this skill does not do

It never creates a cart, opens a checkout, or takes payment. Bombas requires explicit
buyer approval before any payment completes — that flow is
`skills/bombas-shop-and-checkout.md`.
