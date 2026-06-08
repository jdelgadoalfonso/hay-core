# HAY-240 — Wix eCommerce plugin (phase one: API key)

## Goal & acceptance criteria (from HAY-240)

A Wix Stores merchant on a Hay trial connects their store via a pasted API key so the AI can:

- [x] Connect with a scoped API key + site id; connection persists (API keys are long-lived → no token refresh needed in phase one).
- [x] Retrieve an order and its transaction history by order reference (number/id/email).
- [x] Read order fulfilment and status.
- [x] Issue a full or partial refund where the provider supports it; where it doesn't, record an **external** refund and surface a clear fallback message instead of failing silently.
- [x] Failure modes (unsupported provider, expired/invalid key, missing permission) return readable errors that map to a playbook fallback.
- [x] Scope no broader than Shopify/WooCommerce: orders, transactions/refunds, fulfilments, catalog read.

## Verification (done)

- `npm run build --workspace=plugins/core/wix` → clean (strict TS against built @hay/plugin-sdk).
- `mcp/`: `node --check` all files; stdio MCP handshake (initialize + listTools) serves 11 tools.
- git: only source files tracked; dist/ + node_modules gitignored.
- Ships `thumbnail.svg`. Core thumbnail resolution extended to svg > png > jpg
  (`server/services/plugin-asset.service.ts`); server typecheck clean; priority logic unit-tested.

> Phase two (OAuth app on the Wix App Market) is explicitly deferred per the ticket. Documented in README.

## Wix API reference (base host https://www.wixapis.com)

Auth headers: `Authorization: <API_KEY>` (raw, no Bearer) + `wix-site-id: <SITE_ID>`.

- Orders: POST /ecom/v1/orders/search ; GET /ecom/v1/orders/{id}
- Transactions: GET /ecom/v1/orders/{orderId}/transactions
- Refundability: POST /ecom/v1/order-billing/get-order-refundability (refundable | manuallyRefundable | nonRefundable)
- Refund: POST /ecom/v1/order-billing/refund-payments (paymentRefunds[].{transactionId,amount,externalRefund}, sideEffects)
- Fulfilments: GET /ecom/v1/orders/{orderId}/fulfillments ; POST /ecom/v1/fulfillments ; PATCH/DELETE /ecom/v1/fulfillments/{id}
- Catalog v3: POST /stores/v3/products/search ; GET /stores/v3/products/{productId}

## Assumptions to verify against a live store

- Refund money shape assumed `amount: { amount: "<decimal-string>" }`; encapsulated in refunds.js, easy to change if a live call rejects it.
- Search Orders body assumed top-level `{ filter, sort, cursorPaging }` (Wix MongoDB-style filter).
- thumbnail.jpg is the one asset NOT generated here — needs a real Wix icon before ship.
