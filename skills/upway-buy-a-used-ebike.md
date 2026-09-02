---
name: upway-buy-a-used-ebike
description: >-
  Find a certified pre-owned electric bike on Upway's US storefront and take it from
  search to a completed, buyer-approved purchase over Upway's UCP MCP endpoint.
api: Upway Commerce (UCP MCP) API
endpoint: https://upway.co/api/ucp/mcp
transport: mcp
operations:
  - search_catalog
  - get_product
  - create_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-09-02'
method: generated
source: mcp/upway-mcp-tools.json
---

# Buy a certified pre-owned e-bike from Upway

Every tool name and every parameter below is taken from the live `tools/list` response
saved at `mcp/upway-mcp-tools.json`. Nothing here is invented. Response shapes are not
published by Upway, so read what comes back rather than assuming field names.

## Before you start

- **Endpoint**: `POST https://upway.co/api/ucp/mcp`, `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`. JSON-RPC 2.0.
- **No credential.** Discovery, catalog, cart and checkout construction are anonymous.
- **`meta` is required on every single call.** Send
  `{"meta": {"ucp-agent": {"profile": "<your agent profile URI>"}}}` alongside the tool
  arguments. A call without it fails schema validation (`-32602`).
- **Money is in minor units.** `{"amount": 129900, "currency": "USD"}` is $1,299.00.
  Divide by 100 before you quote a price to your buyer.
- **Pass buyer context.** Set `context.address_country` and `context.currency` on
  catalog and cart calls; Upway's own instructions say pricing and availability depend
  on it.

## Steps

1. **Search the catalog.** Call `search_catalog` with `catalog.query` (e.g. "cargo
   e-bike"), and narrow with `catalog.filters.categories`,
   `catalog.filters.price.min` / `.max` (minor units), and
   `catalog.filters.available` (defaults to `true` — sale-ready stock only, which is
   what you want for a marketplace of one-off refurbished bikes). Page with
   `catalog.pagination.cursor` and `catalog.pagination.limit`.

2. **Confirm the exact bike.** Call `get_product` with `catalog.id` and, where the
   listing has options, `catalog.selected[]` as `{name, label}` pairs. Upway sells
   individual refurbished units, so re-confirm availability here before you build a
   cart — a result from a search a few minutes old may already be gone.

3. **Build a cart.** Call `create_cart` with `cart.line_items[]`, each entry being
   `{quantity, item: {id: "<Product Variant ID>"}}`. The `item.id` is the **variant**
   id, not the product id. Add `cart.buyer.email`, `cart.buyer.phone_number` and
   `cart.context`. Mutate with `update_cart` (pass the cart `id`), and use `get_cart` to
   read current totals.

4. **Apply a discount code only if the buyer has one.** `cart.discounts.codes[]` is
   case-insensitive and **replaces** any previously submitted codes rather than adding
   to them — send the full list every time. Do not prompt the buyer for a code they did
   not mention.

5. **Convert to a checkout.** Call `create_checkout` with `checkout.cart_id`, or build
   the checkout directly with `checkout.line_items[]`. This computes real totals, taxes
   and discounts and charges nothing.

6. **Set fulfillment.** Call `update_checkout` with the checkout `id` and
   `checkout.fulfillment.methods[]`: a `type`, the `line_item_ids` it covers, and a
   `destinations[]` entry (`first_name`, `last_name`, `street_address`,
   `extended_address`, `address_locality`, `address_region`, `postal_code`,
   `address_country` as ISO 3166-1 alpha-2, `phone_number`). Select the address with
   `selected_destination_id` and the shipping option with
   `groups[].selected_option_id`. Re-read with `get_checkout` and show the buyer the
   final total including shipping and tax.

7. **Attach payment.** Set `checkout.payment.instruments[]`. Each instrument needs
   `id`, `handler_id`, `type`, and `credential`. The valid `handler_id` values come from
   `https://upway.co/.well-known/ucp` → `payment_handlers`: `gpay`, `shopify.card`,
   `shop_pay`. The schema is conditional — an `apple-pay` handler requires
   `billing_address` plus an `apple_pay_token` credential; every other handler requires
   `credential.token` and `credential.type`. Mark the chosen one `selected: true`.

8. **Get explicit approval, then complete.** Call `complete_checkout` with the checkout
   `id`, the `checkout` body, and **`meta.idempotency-key`** — a required field, not an
   optional header. Generate one key per purchase intent and reuse that same key on any
   retry so a network failure cannot double-charge.

   > Upway's agent instructions are explicit: *"Checkout requires human approval. Agents
   > must not complete payment without explicit buyer consent."* If you cannot get
   > contemporaneous approval at the moment of payment, do not call this tool — route the
   > purchase through Shop Pay (`https://shop.app/SKILL.md`) instead.

9. **Confirm the order.** `complete_checkout` returns the order ID. Call `get_order`
   with that id (format `gid://shopify/Order/123`) to read the confirmed order back to
   the buyer.

## Backing out

- Before `complete_checkout`: call `cancel_checkout` (or `cancel_cart`) with the id.
  Nothing has been charged.
- After `complete_checkout`: **there is no reversal tool.** The undo lives in Upway's
  published policy, not in the protocol — free cancellation within 1 hour of purchase,
  a $75 fee after that but before 8:00 AM EST the next business day and before the bike
  ships, and after that a return under the 14-day Test Ride Policy with a $200
  restocking fee (https://upway.co/policies/refund-policy). Treat `complete_checkout` as
  a one-way door and confirm before it, never after.

## Errors

JSON-RPC error member, not `problem+json`. `-32602` invalid params (usually a missing
`meta.ucp-agent.profile` or missing `meta.idempotency-key`), `-32601` unknown method,
`-32603` internal — retry with the same idempotency key. HTTP `429` means you hit the
per-IP rate limit; back off exponentially, since no `Retry-After` is returned. See
`errors/upway-problem-types.yml`.
