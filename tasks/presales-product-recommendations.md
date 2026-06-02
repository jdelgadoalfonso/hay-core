# Presales / Product Recommendations — Architecture Plan

Status: **Draft for review** · Owner: TBD · Target: alpha

## 1. Goal & Acceptance Criteria

Enable a presales flow where a customer describes a problem/need in conversation and the
agent recommends real products from the merchant's catalog, with rich product cards in the
webchat widget. Catalogs are synced from ecommerce platforms (Shopify-first, also
WooCommerce/Magento) via plugins, from custom stores via a public ingestion API, or entered
manually.

**Done when:**

- A merchant can connect a product-capable plugin (or call the ingestion API) and see their
  catalog mirrored in Hay, embedded for semantic search.
- In a presales-intent conversation, the agent retrieves relevant products (semantic + structured
  filters) and the customer sees product cards (image, title, price, link).
- The Products page appears in the dashboard **only** when a product-capable plugin is enabled.
- Product data inherits the same GDPR/org-lifecycle guarantees as documents.

## 2. Core Architectural Decisions (settled)

1. **Products live in core, not in plugins.** A shared `Product` catalog that multiple plugins
   feed. Plugins are **ingestion adapters**, never owners of the data.
2. **Products ≠ Documents.** Dedicated entities + a **dedicated `product_embeddings` table**
   (separate from `embeddings`) so the catalog can scale and be tuned independently, and so
   deletes/cascades stay cleanly partitioned.
3. **Ingestion is via the existing plugin SDK**, extended with a typed `productSource`
   capability declared in the same `defineHayPlugin` definition that already exposes
   `onInitialize` / `onValidateAuth` / `onStart` / `onEnable` / `onDisable`. **Not** MCP tools,
   **not** a separate SDK. MCP tools remain for _live in-conversation actions_ only.
4. **Retrieval stays in core.** Add a `getRelevantProducts()` path to the retrieval layer, gated
   on a presales/shopping intent. We do **not** build the dormant generic `retriever` plugin
   framework. Sync into core, retrieve from core.
5. **Canonical schema leans Shopify**, maps cleanly from Woo/Magento; each adapter owns its
   platform→canonical mapping so core never learns platform specifics.
6. **UI is gated** by a `products` capability on the plugin manifest (double duty: shows the
   Products nav item AND marks the plugin as a sync adapter).

## 3. Data Model

New entities (extend `OrganizationScopedEntity`: id, organizationId, createdAt/updatedAt,
createdBy/updatedBy, metadata). Files under `server/entities/`.

### `Product` (`products`)

| Field                   | Type                               | Notes                                                              |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------------ |
| `externalId`            | text, indexed                      | Shopify GID · Woo `id` · Magento `sku`                             |
| `source`                | text/enum                          | `shopify` \| `woocommerce` \| `magento` \| `custom` \| `manual`    |
| `handle`                | text, indexed                      | slug                                                               |
| `title`                 | text                               |                                                                    |
| `descriptionHtml`       | text                               |                                                                    |
| `descriptionShort`      | text null                          |                                                                    |
| `vendor`                | text null                          |                                                                    |
| `productType`           | text null                          |                                                                    |
| `status`                | enum                               | `active` \| `draft` \| `archived`                                  |
| `tags`                  | text[]                             |                                                                    |
| `categories`            | jsonb `[{externalId,name,slug}]`   | GIN index                                                          |
| `options`               | jsonb `[{name,position,values[]}]` |                                                                    |
| `images`                | jsonb `[{src,alt,position}]`       |                                                                    |
| `currency`              | char(3)                            | store currency                                                     |
| `priceMin` / `priceMax` | numeric(12,2), indexed             | rollup from variants                                               |
| `available`             | boolean, indexed                   | any variant available                                              |
| `searchText`            | text                               | derived embedding input (see §6)                                   |
| `attributes`            | jsonb                              | source-specific leftovers (metafields/meta_data/custom_attributes) |

### `ProductVariant` (`product_variants`)

| Field                        | Type                                  | Notes                                                              |
| ---------------------------- | ------------------------------------- | ------------------------------------------------------------------ |
| `productId`                  | uuid FK → products (onDelete CASCADE) |                                                                    |
| `externalId`                 | text, indexed                         | variant GID · Woo variation id · Magento child sku                 |
| `sku`                        | text null, indexed                    | **never** the idempotency key (optional/non-unique on Shopify/Woo) |
| `barcode`                    | text null                             |                                                                    |
| `title`                      | text                                  | e.g. "Small / Blue"                                                |
| `selectedOptions`            | jsonb `[{name,value}]`                |                                                                    |
| `position`                   | int                                   |                                                                    |
| `price`                      | numeric(12,2), indexed                |                                                                    |
| `compareAtPrice`             | numeric(12,2) null                    |                                                                    |
| `currency`                   | char(3)                               |                                                                    |
| `inventoryQuantity`          | int null                              |                                                                    |
| `inventoryTracked`           | boolean                               |                                                                    |
| `availability`               | enum, indexed                         | `in_stock` \| `out_of_stock` \| `backorder`                        |
| `weightValue` / `weightUnit` | numeric / enum                        |                                                                    |
| `imageSrc`                   | text null                             |                                                                    |
| `attributes`                 | jsonb                                 | per-variant leftovers                                              |

### `product_embeddings` (raw SQL / pgvector, mirrors `embeddings`)

- Columns: `id`, `organization_id`, `product_id` (FK CASCADE), `content` (the `searchText`),
  `embedding vector(1536)`, `metadata jsonb`, timestamps.
- HNSW index, cosine distance. Model: `text-embedding-3-small` (same as documents).
- **Always synthesize ≥1 variant per product** (Woo simple products / standalone items get a
  default variant) so price/availability live consistently on variants.

### Idempotency

- Unique constraint `(source, external_id)` on both `products` and `product_variants`.
- Sync = `INSERT … ON CONFLICT (source, external_id) DO UPDATE`.

Migrations in `server/database/migrations/` (TypeORM QueryRunner). Follow
`DATABASE_CONVENTIONS.md` (snake_case columns, SnakeNamingStrategy).

## 4. Canonical Type (the adapter contract)

A shared TS type `CanonicalProduct` (+ `CanonicalVariant`) — the normalized shape every
`productSource` function returns and the ingestion API accepts. This is the single
platform-agnostic boundary.

## 5. Plugin SDK Extension — `productSource`

Extend `server/types/plugin-sdk.types.ts` / `plugin.types.ts`:

- New plugin **type/capability** `"products"`.
- New optional block on the plugin definition, reusing existing auth/config/per-org ctx:
  ```ts
  productSource: {
    ingestAllProducts(ctx, cursor?): Promise<{ products: CanonicalProduct[]; nextCursor?: string }>
    ingestProduct(ctx, externalId): Promise<CanonicalProduct | null>
    updateProduct?(ctx, externalId): Promise<CanonicalProduct | null>   // webhook-driven refresh
    deleteProduct?(ctx, externalId): Promise<void>
    handleWebhook?(ctx, event): Promise<void>                            // optional freshness
  }
  ```
- Core's sync engine calls these typed functions; the adapter owns the platform→canonical
  mapping. Plugin manager (`plugin-manager.service.ts`) registers any enabled plugin declaring
  `products` as a sync adapter.

## 6. Sync Engine

A `product-sync.service.ts` + a scheduled job in `scheduled-jobs.registry.ts`:

- **Initial bulk import** on enable: page through `ingestAllProducts`, normalize, upsert, embed.
- **Webhooks** (`updateProduct` / `handleWebhook`): near-real-time price/stock/create/delete.
- **Periodic full re-sync** (scheduler, singleton job) to catch drift.
- **Embedding lifecycle:** (re)compute `searchText` = title + stripped description + vendor +
  productType + tags + option names + category names; embed only when `searchText` changes
  (skip re-embed on pure price/stock updates — that's why price/stock are columns, not embedded).
- **Deletes:** `productVectorStore.deleteByProductIds(orgId, ids)` then delete rows (explicit,
  mirroring document pattern; FK cascade as safety net).

## 7. Public Ingestion API (custom stores)

tRPC routes under `server/routes/v1/products/` (+ REST shim if needed), authenticated by API key:

- `upsertProducts(CanonicalProduct[])` — bulk, idempotent on `(source='custom', external_id)`
- `upsertProduct`, `deleteProduct`
- Optional CSV import for non-technical merchants (later).

Same normalization + embedding path as plugin sync — one contract, many sources.

## 8. Retrieval & Recommendation

- **Intent gating:** extend the perception layer to flag presales/shopping intent. Only then run
  product retrieval (avoids latency/context pollution on support chats).
- **`getRelevantProducts(messages, organizationId, filters?)`** in `retrieval.layer.ts`:
  embed the customer's stated need → vector search over `product_embeddings` (HNSW cosine,
  tuned threshold) + structured pre-filters (budget→price range, "in stock"→availability,
  category). Returns **structured** `Product`+top variant objects.
- **Execution layer** receives structured product objects and reasons over them ("given problem
  X and budget Y, these fit because…"), emitting a product-recommendation response.

## 9. Presentation — Product Card Message Type

- A structured `product_recommendation` message content type (image, title, price,
  compare-at, deep-link button; carousel for multiples) — **not** plain text.
- Rendered in both the **webchat widget** (`/webchat`) and the **dashboard** conversation view.
- Orchestrator emits the structured payload; renderers handle layout.

## 10. Dashboard — Products Page

- `AppSidebar.vue` `navMain`: add a Products item gated by
  `appStore.enabledPlugins.some(p => p.type?.includes('products'))` (mirrors the channels pattern).
- New `dashboard/pages/products.vue`: list/search catalog, view sync status, per-plugin
  product-specific instructions, manual add/edit. Route guarded via middleware.
- Read `.claude/FRONTEND.md` before building pages; use shadcn-vue `Page`/`Card`/`Input` etc.

## 11. GDPR / Privacy Parity

Products are **org business data, not data-subject PII** — so customer erasure/export flows do
**not** touch products. What's required is **org-lifecycle** parity (mirror `privacy.service.ts`):

- `productVectorStore.deleteByProductIds` + reuse org-scoped `deleteByOrganizationId` so org
  deletion purges products + product_embeddings.
- Include products in **org** data export.
- Audit-log `product.created/updated/deleted` + `product.embeddings.generated`.
- Hard-delete embeddings (no soft-delete), matching current `Embedding` behavior.

## 12. Implementation Phases (thin vertical slices)

- [ ] **P1 — Data layer:** `Product` + `ProductVariant` entities, `product_embeddings` table,
      migrations, `product-vector-store.service` (upsert/search/delete, raw SQL). _Verify:_ migration
      runs; unit test upsert+search round-trip.
- [ ] **P2 — Canonical type + ingestion API:** `CanonicalProduct` type, `product-sync.service`
      normalize+upsert+embed, public `products` tRPC routes with API-key auth. _Verify:_ POST sample
      catalog → rows + embeddings exist; idempotent re-POST updates, no dupes.
- [ ] **P3 — SDK `productSource` + sync engine + Shopify adapter:** extend SDK types, plugin
      manager registration, scheduled re-sync job, build the **Shopify** reference adapter (new
      plugin, GraphQL Admin API). _Verify:_ enable plugin → catalog mirrors; webhook updates price.
- [ ] **P4 — Retrieval + intent gating:** perception presales-intent flag, `getRelevantProducts`,
      execution wiring. _Verify:_ presales message returns relevant products; support message does not
      trigger product retrieval.
- [ ] **P5 — Product card message type:** orchestrator payload + webchat + dashboard renderers.
      _Verify:_ recommendation renders as cards with working links in widget.
- [ ] **P6 — Products page + nav gating:** sidebar gate, `products.vue`, route guard, per-plugin
      instructions. _Verify:_ page hidden with no product plugin; visible + functional when enabled.
- [ ] **P7 — GDPR parity:** delete cascade, org export inclusion, audit logging. _Verify:_ org
      deletion purges products+embeddings; export contains products; audit entries written.

## 13. Open Questions (with recommended defaults)

1. **Per-location inventory?** Default: single `inventoryQuantity` column; defer Shopify
   `inventoryLevels` / Magento MSI to a `variant_inventory_levels` child table if needed.
2. **Live price/stock re-fetch for shortlisted items?** Default: use cached catalog for v1; add
   optional live re-fetch of the final 2–3 via adapter in v2.
3. **Recommendation granularity?** Default: product-level (variant text already in product embed).
4. **Embedding table reuse vs dedicated?** Decided: **dedicated** `product_embeddings`.
5. **First reference adapter — Shopify or extend WooCommerce?** Decided: **Shopify** (new
   plugin, priority platform; canonical schema is already Shopify-shaped).

## 14. Verification Story (overall)

`npm run typecheck` + `npm run lint`; server Jest unit tests for vector store + sync normalization;
integration test for the ingestion API; manual presales conversation E2E in the webchat widget.
Each phase ships behind the `products` capability gate, so nothing is user-visible until a
product plugin is enabled — a natural feature flag.
