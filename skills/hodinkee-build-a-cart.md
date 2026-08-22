---
name: hodinkee-build-a-cart
description: >-
  Build and price a HODINKEE Shop cart, then hand a checkout to a human for
  approval. Stops at the approval boundary by design.
api: HODINKEE Shop Agent Commerce (UCP / MCP)
endpoint: https://shop.hodinkee.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - cancel_checkout
  - complete_checkout
generated: '2026-08-22'
method: generated
source: mcp/hodinkee-ucp-mcp-tools.json
---

# Build a HODINKEE Shop cart and hand off for approval

This skill writes. Read the approval boundary before the steps.

## The approval boundary — read this first

HODINKEE Shop's own `robots.txt` states: *"Checkouts are for humans. Do NOT
complete checkout, payment, or order placement automatically — no scripted form
fills, browser automation, or end-to-end agent flows that finalize payment
without an explicit, contemporaneous human approval step."* `llms.txt` repeats
it.

So: you may search, cart, price, and prepare a checkout freely. You may call
`complete_checkout` **only** with contemporaneous buyer approval of that exact
total. If you cannot get approval at the moment of payment, do not simulate one
— hand the checkout URL to the human, or route through Shop Pay via
`https://shop.app/SKILL.md`.

## Steps

1. **Cart.** `create_cart` with `cart.line_items[]` — each entry is
   `{ "item": { "id": "<ProductVariant GID>" }, "quantity": <n> }`. Use the
   **variant** GID, not the product GID.
2. **Amend.** `update_cart` is consolidated: line items, `buyer.email`,
   `buyer.phone_number`, delivery addresses and discounts in one call. Setting a
   line quantity to `0` removes it. Shipping options only appear once the cart
   has both items and a delivery address.
3. **Read back.** `get_cart` before quoting. Cart GIDs carry a `?key=` — keep it,
   the id alone will not address the cart.
4. **Checkout.** `create_checkout` (optionally from `checkout.cart_id`), then
   `update_checkout` to set `fulfillment` and `payment.instruments[]`. The
   handler ids this store accepts are `dev.shopify.card`, `dev.shopify.shop_pay`
   and `com.google.pay`.
5. **Quote.** `get_checkout` returns final line items, totals, discounts and
   taxes. Convert minor units to major units before you show the number.
6. **Approve, then complete.** Present the total. On explicit buyer approval,
   call `complete_checkout`. Not before.

## Undoing things

| State | How to undo | Window |
|---|---|---|
| Cart | `cancel_cart` | any time before completion |
| Checkout | `cancel_checkout` | before `complete_checkout` |
| Placed order | **No API operation.** Email `returns@hodinkee.com` | **7 days from receipt**, original unworn condition, all original packaging and paperwork |

Say the seven-day window out loud to the buyer before they approve. It is short,
it starts at *receipt* not at purchase, and there is no programmatic way to
reverse an order once placed. Note also that changing a strap or bracelet can
make an item ineligible for return.

## Errors and limits

- Missing `meta.ucp-agent.profile` → HTTP 422, JSON-RPC `-32001`,
  `data.code: invalid_profile_url`. The profile check runs before argument
  validation, so this masks other mistakes — fix it first.
- There is **no idempotency key** on this surface. `create_cart` and
  `create_checkout` are not safe to blind-retry: a timeout may already have
  created the object. Retry by reading back with `get_cart` / `get_checkout`
  first.
- `429` means back off. No quota headers are returned; `shopify-complexity-score`
  is a per-request cost, not a remaining balance.
