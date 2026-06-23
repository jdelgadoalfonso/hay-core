I now have all the grounding I need. I have confirmed: the dual-mode auth registration API (`register.auth.apiKey` + `register.auth.oauth2` can both be registered), `ctx.auth.get()` returns `{ methodId, credentials }` where OAuth gives `accessToken`, `config.field()` for refs, and the Twenty MCP tool-file structure. Here is the complete plan.

---

# HAY-219 — Managed Shopify OAuth + Full Admin API Tool Surface

**Branch context:** builds on HAY-221 (`plugins/core/shopify` already ships self-hosted client-credentials auth + token-refresh cron + a single `shopify_get_shop` MCP tool).
**Goal:** add the **managed** (hay.chat App Store) OAuth flow alongside the existing **self-hosted** mode, and expand the MCP from 1 tool to the full Orders / Customers / Products / Fulfillments / Inventory surface (read + the write ops: cancel, refund, address update, note update).
**Admin API version:** `2026-04` (already the plugin default).

---

## 1. Dual-Mode Auth Design

### 1.1 The two modes

| | Self-hosted (HAY-221, exists) | Managed (HAY-219, new) |
|---|---|---|
| Who creates the Shopify app | The customer, in their own Shopify Dev Dashboard | hay.chat (one App Store app, shared) |
| Credentials source | Customer pastes `clientId` + `clientSecret` into plugin config | `SHOPIFY_OAUTH_CLIENT_ID` / `SHOPIFY_OAUTH_CLIENT_SECRET` server env (hay.chat-owned) |
| Grant | `client_credentials` (24h token, refreshed by cron) | `authorization_code` (per-merchant install, long-lived offline token) |
| Token lifetime | 24h, cron-refreshed every 20h | Non-expiring offline access token (no refresh needed) |
| Auth method id | `shopify-credentials` (apiKey-style) | `shopify-oauth` (oauth2) |
| `ctx.auth.get()` shape | reads config → runs grant in `onStart` | `{ methodId: "shopify-oauth", credentials: { accessToken } }` |

### 1.2 Chosen approach: **register BOTH auth methods, select by `authMode` config field**

The SDK already supports registering multiple auth methods in one `onInitialize` (`RegisterAuthAPI` exposes both `apiKey()` and `oauth2()`; `AuthState.methodId` disambiguates at runtime). We exploit this directly.

Add an `authMode` config field (`select`, default `"oauth"`):

- `"oauth"` → **managed**. Client id/secret resolve from `SHOPIFY_OAUTH_CLIENT_ID` / `SHOPIFY_OAUTH_CLIENT_SECRET` env vars (declared via the `env` property on the `clientId`/`clientSecret` config fields). The dashboard auto-renders the Connect button (see §3). `onStart` reads the offline token from `ctx.auth.get()`.
- `"self_hosted"` → **client-credentials** (current HAY-221 behaviour). Customer supplies `clientId`/`clientSecret` in config; the cron mints/refreshes the 24h token; `onStart` runs the grant.

**Why this approach (vs. a single auth method, or two separate plugins):**

- **Register both, branch in `onStart`/`onValidateAuth` on `authMode`** — chosen. One plugin, one MCP, one tool surface. The dashboard already keys the Connect button off the presence of an `oauth2` entry in `metadata.authMethods` + configured client id/secret (Report 2), so registering `oauth2` makes Connect appear automatically. The cost is `onStart` must branch on `authMode`; acceptable and explicit.
- **Single auth method, pick at runtime** — rejected. The dashboard's Connect button visibility and the core OAuth service both read static `metadata.authMethods`; you cannot make one entry behave as both apiKey and oauth2.
- **Two separate plugins (`shopify` + `shopify-managed`)** — rejected. Duplicates the entire MCP tool surface and violates the repo's "no redundant code" rule.

**Trade-off to call out:** the customer sees both an `authMode` selector and the client-id/secret fields. We mark `clientId`/`clientSecret` as **not required at the schema level** and instead validate them conditionally in `onValidateAuth` based on `authMode` (managed mode needs neither in config — they come from env). UI nicety (hiding the self-hosted fields when `authMode=oauth`) is a follow-up via a `before-settings` Vue extension; not blocking for HAY-219.

### 1.3 New config schema (`onInitialize`)

```ts
ctx.register.config({
  authMode: {
    type: "select",
    label: "Connection mode",
    description:
      "Managed: connect with one click via hay.chat's Shopify app. " +
      "Self-hosted: use your own Shopify app's Client ID and secret.",
    options: [
      { value: "oauth", label: "Managed (recommended)" },
      { value: "self_hosted", label: "Self-hosted (own app)" },
    ],
    default: "oauth",
    required: true,
  },
  shopDomain: {
    type: "string",
    label: "Store domain",
    description: "Your myshopify.com domain, e.g. mystore.myshopify.com (without https://).",
    placeholder: "mystore.myshopify.com",
    required: true, // needed in BOTH modes (per-shop authorize URL + token exchange host)
  },
  clientId: {
    type: "string",
    label: "Client ID (self-hosted only)",
    description: "Self-hosted mode only. From your Shopify app's API credentials.",
    required: false,
    env: "SHOPIFY_OAUTH_CLIENT_ID", // managed-mode fallback
  },
  clientSecret: {
    type: "string",
    label: "Client secret (self-hosted only)",
    description: "Self-hosted mode only.",
    required: false,
    encrypted: true,
    env: "SHOPIFY_OAUTH_CLIENT_SECRET", // managed-mode fallback
  },
  apiVersion: {
    type: "string",
    label: "Admin API version",
    default: "2026-04",
  },
});
```

> Verify the SDK `ConfigFieldDescriptor` supports `type: "select"` with `options`. If not, fall back to `type: "string"` with a documented enum and validate in `onValidateAuth`. (Flagged in §8.)

### 1.4 Auth registration (`onInitialize`) — register both

```ts
// Managed: one-click OAuth. Client id/secret resolve from env in managed mode.
ctx.register.auth.oauth2({
  id: "shopify-oauth",
  label: "Connect with Shopify",
  authorizationUrl: "https://{shop}/admin/oauth/authorize",
  tokenUrl: "https://{shop}/admin/oauth/access_token",
  scopes: [
    "read_orders", "write_orders",
    "read_customers", "write_customers",
    "read_products",
    "read_inventory",
    "read_fulfillments", "write_fulfillments",
  ],
  clientId: globalCtx.config.field("clientId"),
  clientSecret: globalCtx.config.field("clientSecret"),
});

// Self-hosted: existing client-credentials path. Validating the secret runs the grant.
ctx.register.auth.apiKey({
  id: "shopify-credentials",
  label: "Self-hosted app credentials",
  configField: "clientSecret",
});

// Cron stays — only relevant in self-hosted mode (managed offline tokens don't expire).
ctx.register.cron({ name: "refresh_shopify_token", schedule: "0 */20 * * *", handler: refreshTokenHandler, retryPolicy: { maxRetries: 3, backoff: "exponential" } });
```

---

## 2. Managed OAuth: the per-shop URL problem + core changes

### 2.1 The problem (confirmed in Report 1 & 2)

`register.auth.oauth2.authorizationUrl` and `tokenUrl` are **static strings** baked into `metadata.authMethods` at initialize time. Shopify is per-merchant: both the authorize and token endpoints live on `https://{shop}/...`. The core `oauthService.initiateOAuth` (`server/services/oauth.service.ts:107-263`) builds the URL by appending query params to the static `authorizationUrl` — it does **not** substitute placeholders.

### 2.2 Required core change: `{shop}` placeholder substitution in the OAuth service

This is the cleanest fix and the one the plan commits to. In `server/services/oauth.service.ts`, in **both** `initiateOAuth` (around the URL-build, ~lines 224-249) and `handleCallback`/`exchangeCodeForTokens` (token URL, ~lines 459-492), resolve `{shop}` from the instance's `shopDomain` config field before use:

```ts
// helper, used in both initiate + callback
function resolveShopPlaceholder(url: string, instanceConfig: Record<string, unknown>): string {
  if (!url.includes("{shop}")) return url;
  const shopRaw = String(instanceConfig?.shopDomain ?? "");
  const shop = shopRaw.trim().replace(/^https?:\/\//i, "").replace(/\/+$/, "");
  if (!shop) throw new Error("Shopify connection requires a store domain before connecting.");
  return url.replace(/\{shop\}/g, shop);
}
```

- `initiateOAuth`: apply to `authorizationUrl` before appending params.
- `handleCallback`: apply to `tokenUrl` before the POST.

**Why core (not plugin):** the OAuth flow is owned by core; the plugin only declares descriptors. The `{shop}` template + instance-config substitution is a small, generic, reusable primitive (any per-tenant OAuth provider benefits). It is **breaking-change-safe** — URLs without `{shop}` pass through untouched.

> **Alternative considered & rejected:** Shopify's domain-agnostic `https://shopify.com/admin/oauth/authorize?shop=...` (Report 1 option). It still needs the `shop` query param injected per-instance, so it does not avoid a core change, and the token endpoint is still per-shop. Placeholder substitution is strictly more general. **Flagged for dev-store verification** (§8): confirm `https://{shop}/admin/oauth/authorize` is the correct authorize host for an App-Store/custom-distribution app and that Shopify redirects back to our `${API_URL}/oauth/callback` (must be whitelisted in the app's allowed redirect URLs).

### 2.3 Token storage (no plugin work)

`handleCallback` → `storeTokens` already persists `{ methodId: "shopify-oauth", credentials: { accessToken, refreshToken?, expiresAt?, ... } }` via the encrypted transformer (Report 1). Shopify managed offline tokens are non-expiring and return no refresh token — fine; `expiresAt` is simply absent.

### 2.4 `redirect_uri` whitelist

The core redirect URI is `${getApiUrl()}/oauth/callback` (Report 1, `oauth.service.ts:32`). This exact URL must be added to the hay.chat Shopify app's **Allowed redirection URL(s)** in the Shopify Partner/Dev dashboard. **Manual config step, document in README.**

---

## 3. Dashboard Connect/Disconnect

Per Report 2, **no plugin Vue code is required** for the button to appear. The Connect/Disconnect control in `dashboard/components/plugins/PluginOAuthConnection.vue` renders automatically when `enabled && oauthAvailable && oauthConfigured`:

- `oauthAvailable` ← `metadata.authMethods` contains an `oauth2` entry → satisfied by registering `shopify-oauth` (§1.4).
- `oauthConfigured` ← resolved `clientId` + `clientSecret` both have values → satisfied in managed mode by the `SHOPIFY_OAUTH_CLIENT_ID/SECRET` env vars (config-resolver env fallback, Report 1 §4).

**Connect** → `Hay.plugins.oauth.initiate.mutate({ pluginId })` (`plugins.handler.ts:1404`). **Disconnect** → `Hay.plugins.oauth.revoke.mutate({ pluginId })` (`plugins.handler.ts:1493`).

**Ordering constraint (UX):** `shopDomain` must be saved **before** Connect, because the `{shop}` substitution reads it. Two options:
1. (Minimal, chosen) The `shopDomain` config field is required and saved via the normal settings form first; the resolver throws a clear error if Connect is clicked with no domain. Document the order.
2. (Follow-up) A `before-settings` Vue extension that gates the Connect button until `shopDomain` is set. Not blocking.

**Server env:** add `SHOPIFY_OAUTH_CLIENT_ID` and `SHOPIFY_OAUTH_CLIENT_SECRET` to `.env.example` and the deploy secrets.

---

## 4. `onStart` — token acquisition in BOTH modes

```ts
async onStart(ctx) {
  const apiVersion = ctx.config.getOptional<string>("apiVersion") || "2026-04";
  const shopRaw = ctx.config.getOptional<string>("shopDomain");
  if (!shopRaw) { ctx.logger.info("Shopify: no store domain yet — MCP not started."); return; }
  const shop = normalizeShopDomain(shopRaw);
  const mode = ctx.config.getOptional<string>("authMode") || "oauth";

  let accessToken: string | undefined;

  if (mode === "oauth") {
    // MANAGED: token was minted by core's authorization_code exchange and stored.
    const authState = ctx.auth.get();
    if (authState?.methodId === "shopify-oauth") {
      accessToken = String(authState.credentials.accessToken ?? "") || undefined;
    }
    if (!accessToken) {
      ctx.logger.info("Shopify managed mode: not connected yet (no access token). MCP not started.");
      return;
    }
  } else {
    // SELF-HOSTED: run the client-credentials grant (existing HAY-221 path).
    const creds = readCredentials(ctx);
    if (!creds) { ctx.logger.info("Shopify self-hosted: credentials missing. MCP not started."); return; }
    try {
      const token = await clientCredentialsGrant(creds.shopDomain, creds.clientId, creds.clientSecret);
      accessToken = token.accessToken;
    } catch (e) { ctx.logger.error("Shopify self-hosted token grant failed", e); return; }
  }

  await ctx.mcp.startLocalStdio({
    id: "shopify-mcp",
    command: "node",
    args: ["index.js"],
    cwd: "./mcp",
    env: { SHOPIFY_SHOP: shop, SHOPIFY_ACCESS_TOKEN: accessToken, SHOPIFY_API_VERSION: apiVersion },
  });
}
```

**`onValidateAuth`** branches the same way: managed mode → require `shopDomain` + a stored `shopify-oauth` token (or just `shopDomain` pre-connect); self-hosted → require `shopDomain`+`clientId`+`clientSecret` and run the grant (existing logic). The cron `refreshTokenHandler` should early-return when `authMode === "oauth"`.

The MCP server itself is **mode-agnostic** — it only ever sees `SHOPIFY_SHOP` + `SHOPIFY_ACCESS_TOKEN` + `SHOPIFY_API_VERSION`. No MCP changes for dual mode.

---

## 5. MCP Tool Surface

Refactor the single-file `mcp/index.js` to the Twenty multi-file structure. New files under `plugins/core/shopify/mcp/`:

```
mcp/
  index.js              # rewritten: require + register each tool module
  lib/client.js         # shopifyGql() GraphQL client w/ retry (Report 3)
  lib/format.js         # ok/fail/pageInfo/unwrapConnection/buildFilter (Report 3)
  tools/shop.js         # shopify_get_shop (moved from index.js)
  tools/orders.js       # orders read + cancel/refund/note (Report 4)
  tools/fulfillments.js # tracking / WISMO (Report 4)
  tools/customers.js    # customers read/update + address create/update (Report 5)
  tools/products.js     # products read/search (Report 6)
  tools/inventory.js    # inventory check by variant/SKU (Report 6)
```

`mcp/index.js` mirrors Twenty: `require` each `register*Tools`, register on one `McpServer({ name: "shopify" })`, connect stdio.

`lib/client.js` and `lib/format.js`: use the skeletons from Report 3 verbatim (native `fetch`, `X-Shopify-Access-Token` header, retry on 429/5xx, GraphQL-errors-on-200 handling, `unwrapConnection`/`pageInfo`). Tools use `server.tool(name, description, zodShape, handler)` and wrap with `ok()`/`fail()`.

### 5.1 `tools/orders.js`

| Tool | Input (zod) | GraphQL op |
|---|---|---|
| `shopify_get_order` | `order_id: string` (GID) | `query order(id:ID!){ id name email createdAt displayFinancialStatus displayFulfillmentStatus note totalPriceSet{shopMoney{amount currencyCode}} customer{id displayName email} lineItems(first:100){edges{node{id title quantity sku}}} }` |
| `shopify_get_order_by_name` | `name: string` (`#1001`) | `query orderByIdentifier(identifier:{name:$name}){ id name displayFinancialStatus displayFulfillmentStatus note }` |
| `shopify_get_order_status` | `order_id: string` | `query order(id:$id){ id name displayFinancialStatus displayFulfillmentStatus }` |
| `shopify_list_customer_orders` | `customer_id: string`, `limit?: int=10` | `query customer(id:$id){ orders(first:$first sortKey:CREATED_AT reverse:true){ edges{node{ id name createdAt displayFinancialStatus displayFulfillmentStatus totalPriceSet{shopMoney{amount currencyCode}} }}} }` |
| `shopify_search_orders` | `query?`, `limit?=20`, `cursor?`, `financialStatus?`, `fulfillmentStatus?`, `createdAfter?`, `createdBefore?` | `query orders(first:$first after:$after query:$query){ edges{node{...}cursor} pageInfo{hasNextPage endCursor} }` (build `$query` from filters) |
| `shopify_cancel_order` **(write)** | `order_id`, `reason: enum(CUSTOMER\|DECLINED\|FRAUD\|INVENTORY\|OTHER\|STAFF)`, `restock: bool`, `refund_to_original_payment?: bool`, `notify_customer?: bool`, `staff_note?: string` | `mutation orderCancel(orderId:$orderId reason:$reason restock:$restock refundMethod:$refundMethod notifyCustomer:$notify staffNote:$note){ job{id done} orderCancelUserErrors{field message code} userErrors{field message} }` |
| `shopify_create_refund` **(write)** | `order_id`, `note?`, `notify?`, `refund_line_items?: [{lineItemId,quantity,restockType?}]`, `shipping_amount?`, `transactions?: [{parentId,amount,gateway,kind}]`, `idempotency_key: string` | `mutation refundCreate(input:$input) @idempotent(key:$key){ order{id name} refund{id totalRefundedSet{shopMoney{amount currencyCode}}} userErrors{field message} }` |
| `shopify_add_order_note` **(write)** | `order_id`, `note: string` | `mutation orderUpdate(input:{id:$id note:$note}){ order{id note} userErrors{field message} }` |

All tools normalize bare numeric ids to `gid://shopify/Order/{id}`. Every mutation surfaces `userErrors` as a thrown error via `fail()`.

### 5.2 `tools/fulfillments.js`

| Tool | Input | GraphQL |
|---|---|---|
| `shopify_get_order_tracking` (WISMO) | `order_id: string` | `query order(id:$id){ id name displayFulfillmentStatus fulfillments(first:10){ status displayStatus estimatedDeliveryAt trackingInfo{number url company} } }` |

### 5.3 `tools/customers.js`

| Tool | Input | GraphQL |
|---|---|---|
| `shopify_find_customer_by_email` | `email: string`, `first?: int=10` | `query customers(first:$first query:$query){ edges{node{ id firstName lastName defaultEmailAddress{emailAddress} defaultPhoneNumber{phoneNumber} numberOfOrders }} pageInfo{hasNextPage endCursor} }` — `$query = email:"..."` |
| `shopify_get_customer_by_identifier` | `emailAddress?`, `phoneNumber?` (E.164) | `query customerByIdentifier(identifier:$identifier){ id firstName lastName defaultEmailAddress{emailAddress} defaultPhoneNumber{phoneNumber} numberOfOrders }` |
| `shopify_get_customer` | `customerId: string` | `query customer(id:$id){ id firstName lastName defaultEmailAddress{emailAddress marketingState} defaultPhoneNumber{phoneNumber} numberOfOrders amountSpent{amount currencyCode} note tags createdAt defaultAddress{ id address1 address2 city province provinceCode country countryCode zip firstName lastName phone company } }` |
| `shopify_update_customer` **(write)** | `customerId`, `firstName?`, `lastName?`, `email?`, `phone?`, `note?`, `tags?: [string]` | `mutation customerUpdate(input:$input){ customer{ id firstName lastName defaultEmailAddress{emailAddress} defaultPhoneNumber{phoneNumber} note tags } userErrors{field message} }` (note: `CustomerInput.email`/`phone` are the plain string fields, NOT deprecated) |
| `shopify_create_customer_address` **(write)** | `customerId`, `address: {address1,address2,city,province,provinceCode,country,countryCode,zip,firstName,lastName,phone,company}`, `setAsDefault?: bool` | `mutation customerAddressCreate(customerId:$customerId address:$address setAsDefault:$setAsDefault){ address{id address1 city province country zip} userErrors{field message} }` |
| `shopify_update_customer_address` **(write)** | `customerId`, `addressId`, `address: {...}`, `setAsDefault?: bool` | `mutation customerAddressUpdate(customerId:$customerId addressId:$addressId address:$address setAsDefault:$setAsDefault){ address{id address1 city province country zip} userErrors{field message} }` |

> Read side uses `defaultEmailAddress`/`defaultPhoneNumber` (top-level `email`/`phone` deprecated on the Customer object in 2026-04). Pass address GIDs through verbatim from `customerAddressCreate`/`customer.addresses` — do not hand-construct (§8).

### 5.4 `tools/products.js`

| Tool | Input | GraphQL |
|---|---|---|
| `shopify_get_product` | `id: string`, `variantsFirst?: int=50` | `query product(id:$id){ id title handle status description totalInventory priceRangeV2{minVariantPrice{amount currencyCode} maxVariantPrice{amount currencyCode}} variants(first:$variantsFirst){ nodes{ id title sku price compareAtPrice availableForSale inventoryQuantity selectedOptions{name value} } } }` |
| `shopify_get_product_by_handle` | `handle: string`, `variantsFirst?: int=50` | `query productByIdentifier(identifier:{handle:$handle}){ ...same fields... }` (`productByHandle` deprecated) |
| `shopify_search_products` | `title: string`, `first?: int=25`, `sortKey?: enum=RELEVANCE`, `after?` | `query products(first:$first query:$query sortKey:$sortKey after:$after){ nodes{ id title handle status totalInventory priceRangeV2{...} } pageInfo{hasNextPage endCursor} }` — build `$query = title:*${title}*` |

### 5.5 `tools/inventory.js`

| Tool | Input | GraphQL |
|---|---|---|
| `shopify_check_inventory` | `variantId?` **xor** `sku?`, `quantityNames?: [enum]=["available","on_hand","committed"]` | by id: `query productVariant(id:$variantId){ id sku price availableForSale inventoryQuantity inventoryItem{ id tracked inventoryLevels(first:20){ nodes{ location{id name} quantities(names:$quantityNames){name quantity} } } } }`; by SKU: `query productVariants(first:1 query:$query){ nodes{...same...} }` with `$query = sku:${sku}` |

> `productVariantByIdentifier` does **not** accept SKU in 2026-04 — SKU path MUST use `productVariants(query:"sku:...")` (Report 6). `quantities(names:)` is required (`[String!]!`) — always pass at least `["available"]`.

---

## 6. `mcp/package.json`

Add `zod` to `plugins/core/shopify/mcp/package.json` dependencies (tools use `z` for input schemas, mirroring Twenty). Keep `@modelcontextprotocol/sdk`. Run `npm install` in `mcp/` and rebuild the plugin bundle.

---

## 7. Ordered Task Checklist

1. **Config + auth (src/index.ts):** add `authMode` + relax `clientId`/`clientSecret` to optional with `env` fallbacks; register `auth.oauth2("shopify-oauth", authorizationUrl/tokenUrl with {shop})` alongside the existing `auth.apiKey`.
2. **`onStart` branch** on `authMode` (managed → `ctx.auth.get()` token; self-hosted → grant). Early-return cron when managed.
3. **`onValidateAuth` branch** on `authMode`.
4. **Core OAuth service:** add `{shop}` placeholder substitution from instance `shopDomain` in `initiateOAuth` (authorize URL) and `handleCallback`/`exchangeCodeForTokens` (token URL) in `server/services/oauth.service.ts`.
5. **Env:** add `SHOPIFY_OAUTH_CLIENT_ID` / `SHOPIFY_OAUTH_CLIENT_SECRET` to `.env.example` + README setup (incl. redirect-URI whitelist note + Connect-after-shopDomain ordering).
6. **MCP scaffold:** create `lib/client.js`, `lib/format.js`; refactor `index.js` to multi-module registration; move `shopify_get_shop` to `tools/shop.js`.
7. **MCP tools:** implement `tools/orders.js`, `fulfillments.js`, `customers.js`, `products.js`, `inventory.js` per §5.
8. **`mcp/package.json`:** add `zod`; `npm install`; rebuild bundle.
9. **Verify (real dev store):** the items in §8.
10. **Lint/typecheck/build:** `npm run typecheck` + plugin build script; smoke-test MCP server boots with a token.

---

## 8. Must-Verify-Against-Real-Dev-Store (flagged uncertainties)

1. **Authorize host for managed app:** confirm `https://{shop}/admin/oauth/authorize` is correct for a hay.chat App-Store/custom-distribution app, and that Shopify accepts `${API_URL}/oauth/callback` as a whitelisted redirect. (§2.2)
2. **`orderCancel.refundMethod` required-ness:** Report 4 found the version-pinned page renders it **optional** but a summary said required. Confirm before shipping `shopify_cancel_order` without it.
3. **`refundCreate` nested input field names:** `RefundLineItemInput.restockType`, `ShippingRefundInput` (`shipping` vs `shipping.amount`), `OrderTransactionInput.kind` enum — validate against 2026-04 input-object pages / GraphiQL.
4. **`@idempotent(key:)` directive:** confirmed required for `refundCreate` in 2026-04; confirm exact directive arg name (`key`).
5. **`Order.fulfillments` arg shape:** confirm `first:` is accepted (vs a plain `[Fulfillment!]!` list) on `Order` in 2026-04.
6. **`defaultEmailAddress` sub-fields:** confirm `marketingState` exists (came from a fast-model summary).
7. **Address GID format** for `customerAddressUpdate` — pass through verbatim, don't construct.
8. **`title:*shirt*` wildcard** product search — validate live.
9. **SDK `type: "select"`** config field support — fall back to string+enum validation if unsupported.
10. **`config.field()` availability in `onInitialize`** (used for the oauth2 client refs) — confirm it's `globalCtx.config.field` per the Twenty/SDK examples.

## Acceptance Criteria

- **Managed:** With only `SHOPIFY_OAUTH_CLIENT_ID/SECRET` in server env and `authMode=oauth`, a user enters `shopDomain`, clicks **Connect**, completes Shopify consent, lands back on the plugin page with status connected; `onStart` reads the stored offline token and the MCP starts; `shopify_get_shop` returns the store. **Disconnect** clears auth and the MCP stops on next start.
- **Self-hosted:** With `authMode=self_hosted` + customer `clientId/clientSecret/shopDomain`, existing HAY-221 behaviour is unchanged (grant + 20h cron) and the same MCP tool surface works.
- **Tools:** every tool in §5 is registered and returns a sane payload (or a clear `userErrors`-derived error) against a dev store; read tools paginate via cursors; write tools (cancel/refund/note/customer-update/address-create/address-update) succeed end-to-end.
- `npm run typecheck` and the plugin build pass; no hardcoded plugin id in core; `{shop}` substitution is generic (non-templated URLs unaffected).

---

**Key files touched:** `plugins/core/shopify/src/index.ts`, `plugins/core/shopify/mcp/{index.js,package.json}`, new `plugins/core/shopify/mcp/lib/{client,format}.js` + `tools/{shop,orders,fulfillments,customers,products,inventory}.js`, `server/services/oauth.service.ts` ({shop} substitution), `.env.example`, `plugins/core/shopify/README.md`. No dashboard code changes required for Connect/Disconnect (renders from metadata).