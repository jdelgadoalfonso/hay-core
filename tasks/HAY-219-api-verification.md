# HAY-219 §8 — Shopify 2026-04 API verification results


## orders.js
- **[corrected]** Query orderByIdentifier(identifier:{name:"#1001"}) — accepts a {name} identifier?
  - fix: The orderByIdentifier query DOES exist in 2026-04, but OrderIdentifierInput has ONLY two fields: id (ID) and customId (UniqueMetafieldValueInput). There is NO 'name' field. To look up an order by its name (e.g. '#1001'), use the orders connection with a query filter instead: orders(first:1, query:"name:#1001"){ edges { node { id name } } }. Use orderByIdentifier only with {id} or {customId:{namespace,key,value}}.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/input-objects/OrderIdentifierInput
- **[confirmed]** Mutation orderCancel — is refundMethod REQUIRED or optional? input type? does orderCancelUserErrors exist?
  - fix: refundMethod is OPTIONAL (type OrderCancelRefundMethodInput). Required args are orderId: ID!, reason: OrderCancelReason!, restock: Boolean!. notifyCustomer and staffNote are optional. The payload field orderCancelUserErrors: [OrderCancelUserError!]! DOES exist (and is preferred over the deprecated userErrors).
  - https://shopify.dev/docs/api/admin-graphql/2026-04/mutations/orderCancel
- **[confirmed]** Mutation refundCreate(input: RefundInput!) — name, @idempotent(key:) requirement, RefundInput / RefundLineItemInput.restockType / ShippingRefundInput / OrderTransactionInput.kind
  - fix: Mutation name refundCreate is correct; single argument is input: RefundInput!. As of 2026-04 the @idempotent(key:) directive is required. RefundLineItemInput.restockType is of enum type RefundLineItemRestockType (nullable); its sibling fields are lineItemId: ID!, quantity: Int!, locationId: ID. ShippingRefundInput uses amount (with fullRefund: Boolean alternative). OrderTransactionInput.kind is of type OrderTransactionKind. Recommend confirming ShippingRefundInput.amount is supplied as a MoneyInput-style value per the field doc.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/mutations/refundCreate
- **[confirmed]** Mutation orderUpdate(input:{id, note}) — OrderInput accepts note in 2026-04?
  - fix: Correct. orderUpdate takes input: OrderInput!. OrderInput requires id: ID! and accepts note: String (overwrites existing note). Both fields are valid in 2026-04.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/input-objects/OrderInput

## products.js
- **[confirmed]** productByIdentifier(identifier:{handle:"x"}) exists and accepts {handle}
  - fix: Valid in 2026-04. The `productByIdentifier` query takes `identifier: ProductIdentifierInput!`, which accepts `handle` (or `customId`/`id`). productByHandle is deprecated; productByIdentifier is the replacement.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/queries/productByIdentifier
- **[corrected]** products(query:"title:*shirt*") surrounding-wildcard partial title match
  - fix: Surrounding/leading wildcards (`*shirt*`, `title:*shirt*`) are NOT supported. Only trailing prefix wildcards work. Use `query: "title:shirt*"` to match titles whose terms begin with "shirt". For broader full-text matching, a bare `query: "shirt"` also searches indexed text fields including title.
  - https://shopify.dev/docs/api/usage/search-syntax
- **[confirmed]** Product fields: title, handle, status, description, totalInventory, priceRangeV2{minVariantPrice/maxVariantPrice{amount currencyCode}}, variants{nodes{id title sku price compareAtPrice availableForSale inventoryQuantity selectedOptions{name value}}}
  - fix: All confirmed in 2026-04. priceRangeV2 is ProductPriceRangeV2 with minVariantPrice/maxVariantPrice (both MoneyV2 non-null, exposing amount + currencyCode). All listed ProductVariant fields (id, title, sku, price, compareAtPrice, availableForSale, inventoryQuantity, selectedOptions{name value}) exist. variants is a connection; `nodes{}` is valid.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/objects/Product

## customers.js
- **[confirmed]** Customer read fields: defaultEmailAddress{emailAddress marketingState}, defaultPhoneNumber{phoneNumber}, numberOfOrders, amountSpent{amount currencyCode}
  - fix: All confirmed. defaultEmailAddress is type CustomerEmailAddress (nullable) with subfields emailAddress and marketingState (CustomerEmailAddressMarketingState enum). defaultPhoneNumber is CustomerPhoneNumber with phoneNumber. numberOfOrders is UnsignedInt64!. amountSpent is MoneyV2! with amount + currencyCode. marketingState path defaultEmailAddress.marketingState is correct.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/objects/Customer
- **[confirmed]** customerByIdentifier(identifier:{emailAddress | phoneNumber})
  - fix: Query exists. Arg identifier is CustomerIdentifierInput! with fields emailAddress (String), phoneNumber (String), and customId (namespace/key/value). Requires read_customers scope.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/queries/customerByIdentifier
- **[confirmed]** customerUpdate(input: CustomerInput) with plain email/phone, firstName/lastName/note/tags
  - fix: CustomerInput has plain String fields email and phone, neither deprecated. firstName, lastName, note, tags all present. (Only the addresses field on CustomerInput is deprecated.)
  - https://shopify.dev/docs/api/admin-graphql/2026-04/input-objects/CustomerInput
- **[confirmed]** customerAddressCreate(customerId, address, setAsDefault) and customerAddressUpdate(customerId, addressId, address, setAsDefault); addressId is a GID
  - fix: Both mutations exist. customerAddressCreate(customerId: ID!, address: MailingAddressInput!, setAsDefault: Boolean). customerAddressUpdate(customerId: ID!, addressId: ID!, address: MailingAddressInput!, setAsDefault: Boolean). Note: customerAddressUpdate DOES require customerId in addition to addressId. addressId is an ID! (GID for MailingAddress).
  - https://shopify.dev/docs/api/admin-graphql/2026-04/mutations/customerAddressUpdate

## fulfillments.js
- **[corrected]** Order.fulfillments — connection accepting (first: N) vs plain list [Fulfillment!]!
  - fix: Order.fulfillments is a PLAIN LIST: `fulfillments(first: Int, query: String): [Fulfillment!]!`. It is NOT a paginated connection — there is no edges/node/pageInfo, and no after/last/before cursor args. `first` simply truncates the array. Select directly: `fulfillments(first: 10) { id status trackingInfo { number url company } }` — do NOT wrap in `edges { node { ... } }`.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/objects/Order
- **[confirmed]** Fulfillment fields: status, displayStatus, estimatedDeliveryAt, trackingInfo{number url company}
  - fix: All confirmed. status: FulfillmentStatus! ; displayStatus: FulfillmentDisplayStatus ; estimatedDeliveryAt: DateTime ; trackingInfo: [FulfillmentTrackingInfo!]! whose subfields are number: String, url: URL, company: String. Note trackingInfo is a non-null LIST, so select it as `trackingInfo { number url company }` (each element); it also accepts an optional `first: Int` arg.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/objects/Fulfillment

## inventory.js
- **[confirmed]** productVariant(id:) fields: sku, price, availableForSale, inventoryQuantity, inventoryItem{id tracked inventoryLevels(first:){nodes{location{id name} quantities(names:$names){name quantity}}}}
  - fix: All confirmed in 2026-04. productVariant(id: ID!) returns ProductVariant. Fields: sku (String), price (Money!), availableForSale (Boolean!), inventoryQuantity (Int), inventoryItem (InventoryItem!). InventoryItem.id (ID!), tracked (Boolean!), inventoryLevels (InventoryLevelConnection!, accepts first/after/etc). InventoryLevel exposes location (Location! with id, name) and quantities. InventoryQuantity exposes name (String!) and quantity (Int!). The nested shape is valid.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/objects/ProductVariant
- **[confirmed]** quantities(names: [String!]!) requires the names arg; valid name values include available, on_hand, committed
  - fix: Confirmed: InventoryLevel.quantities takes a REQUIRED names argument of type [String!]!. Valid inventory state names per Shopify inventory-states docs are: available, incoming, committed, reserved, damaged, safety_stock, quality_control, on_hand (on_hand is a computed/aggregate state). available, on_hand, and committed are all valid.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/objects/InventoryLevel
- **[confirmed]** SKU lookup via productVariants(first:1, query:"sku:ABC"); and productVariantByIdentifier does NOT take SKU
  - fix: Correct on both counts. productVariants query accepts first and query args; query filter supports the sku: search field (e.g. query:"sku:ABC", also wildcard sku:element*). productVariantByIdentifier takes ProductVariantIdentifierInput which has ONLY id and customId fields — no sku field — so it cannot look up by SKU. Use productVariants(first:1, query:"sku:...") for SKU lookup.
  - https://shopify.dev/docs/api/admin-graphql/2026-04/queries/productVariants