---
name: Shop the Bombas catalog and complete a checkout
description: Search the Bombas storefront catalog, build a cart, open a checkout, set
  shipping, and complete the purchase with explicit buyer approval, over the store's
  UCP Shopping MCP endpoint.
api: mcp/bombas-ucp-shopping-2026-04-08.openrpc.json
endpoint: https://shop.bombas.com/api/ucp/mcp
operations:
- search_catalog
- get_product
- create_cart
- update_cart
- create_checkout
- update_checkout
- complete_checkout
- get_order
generated: '2026-07-31'
method: generated
source: https://shop.bombas.com/agents.md
---

# Shop Bombas and complete a checkout

Bombas runs a Shopify Plus storefront that implements the Universal Commerce Protocol.
Everything below uses method names taken verbatim from the OpenRPC contract the store
declares at `/.well-known/ucp`. Bombas publishes no REST API and issues no API keys.

## Before you call anything

1. `GET https://shop.bombas.com/.well-known/ucp` and confirm the version you intend to
   use is in `supported_versions` (`2026-04-08` is current; `2026-01-23` is still served).
2. Every call is a JSON-RPC 2.0 POST to `https://shop.bombas.com/api/ucp/mcp` with
   `Content-Type: application/json` and `Accept: application/json, text/event-stream`.
3. Every call must carry `meta.ucp-agent.profile` — the URI of *your* platform's UCP
   profile document (HTTP header form: `UCP-Agent`). Calling without it returns JSON-RPC
   error `-32001` `UCP discovery failed` / `invalid_profile_url` and nothing else works.
4. Pass buyer context (`context.address_country`, `context.currency`) so prices and
   availability are correct for the buyer.

## Steps

1. **Find products** — call `search_catalog` with the buyer's intent as query text plus
   any filters and pagination. For a specific item, call `get_product`; for a batch of
   known identifiers, call `lookup_catalog`.
2. **Build the cart** — call `create_cart` with the chosen line items, then `update_cart`
   to change quantities or swap variants. `get_cart` re-reads current state.
3. **Open the checkout** — call `create_checkout`. Read `messages[]` on the result; a
   message with severity `requires_buyer_input` means the merchant needs something the
   API cannot collect programmatically and the checkout is not yet completable.
4. **Set fulfillment** — call `update_checkout` with the shipping address and the chosen
   shipping method. This store ships to a **single destination only**
   (`allows_multi_destination.shipping` is false), so do not attempt to split a shipment.
5. **Get buyer approval, then complete** — call `complete_checkout` **only** after the
   buyer has explicitly approved the payment. This is a hard rule in Bombas' own
   `agents.md`: agents must not complete payment without contemporaneous buyer consent.
   If you cannot obtain it in the moment, stop and route the purchase through Shop Pay
   using the Shopify Shop skill (`https://shop.app/SKILL.md`) instead.
6. **Confirm** — call `get_order` for the current-state snapshot of the placed order.

## Rules that will bite you

- **Idempotency is required, not optional, on the destructive calls.** `complete_checkout`,
  `cancel_checkout` and `cancel_cart` all require `meta.idempotency-key` (a UUID, mapped
  to the HTTP `Idempotency-Key` header). Generate one per logical attempt and **reuse the
  same key on every retry** of that attempt. Send one on the other mutating calls too.
- **Back off on 429.** The MCP endpoint is rate limited per IP. Retry with exponential
  backoff; do not parallelize checkout traffic.
- **Errors are not RFC 9457.** Business failures come back as
  `{ucp: {status: error}, messages: [...], continue_url}`. Branch on `messages[].severity`:
  `recoverable` (fix inputs and retry), `requires_buyer_input` / `requires_buyer_review`
  (hand off to the buyer, use `continue_url`), `unrecoverable` (start a new resource).
  `messages[].path` is an RFC 9535 JSONPath to the exact offending component.
- **Payment handlers available on this store:** Shop Pay, Shopify card (visa, master,
  american_express, discover, diners_club) and Google Pay.

## If you only need to read

No authentication is needed for browsing:
`GET /collections/all`, `GET /products/{handle}.json`,
`GET /collections/{handle}/products.json`, `GET /search?q={query}&type=product`,
`GET /sitemap.xml` — all on `https://shop.bombas.com`.
