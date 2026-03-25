# WhatsApp Channel Plugin (Twilio) — Implementation Plan

## Context

We're adding WhatsApp as a communication channel using Twilio (not direct Meta API) for faster time-to-market. The existing plugin system and conversation infrastructure already support channels — the `Conversation.channel` enum includes `"whatsapp"`, the plugin API has a `messages.receive` endpoint, and per-channel agent routing exists. The critical missing piece is **outbound message delivery**: when the orchestrator generates a bot response on a WhatsApp conversation, nothing currently sends it to the customer via Twilio. This plan builds the WhatsApp plugin and the generic channel delivery infrastructure.

**Scope**: Text-only, no media, no read receipts, no embedded phone provisioning. Conversations close when the 24h window expires (handled by existing inactivity timeout). No MCP tools needed — delivery is automatic via the Channel Delivery Service.

---

## How Outbound Delivery Works

```
Orchestrator generates response
  → conversation.addMessage({ type: BOT_AGENT, content: "Hello!" })
    → Saved to PostgreSQL
    → Published to Redis "websocket:events" channel
      ├─ WebSocket Service (existing) → broadcasts to browser clients (webchat)
      └─ Channel Delivery Service (NEW) → checks conversation.channel
           ├─ "web" → skip (WebSocket already handled it)
           └─ "whatsapp" → finds plugin worker → POST /deliver → Twilio API → WhatsApp
```

The orchestrator and `addMessage()` don't change. The Channel Delivery Service is a second Redis subscriber (same pattern as WebSocket service at `server/services/websocket.service.ts:114`).

---

## Phase 1: Plugin Skeleton + Config

Create the WhatsApp plugin following the Zendesk/Stripe plugin pattern.

### New Files

| File                                  | Purpose                                                         |
| ------------------------------------- | --------------------------------------------------------------- |
| `plugins/core/whatsapp/package.json`  | Plugin manifest with `twilio` dependency, `category: "channel"` |
| `plugins/core/whatsapp/tsconfig.json` | TypeScript config (match Zendesk pattern)                       |
| `plugins/core/whatsapp/src/index.ts`  | Main entry: `defineHayPlugin()` with config, auth, routes       |

### Key Details

**Config registration** (in `onInitialize`):

- `accountSid` — `{ type: "string", encrypted: false, env: "TWILIO_ACCOUNT_SID" }`
- `authToken` — `{ type: "string", encrypted: true, env: "TWILIO_AUTH_TOKEN" }`
- `whatsappNumber` — `{ type: "string", encrypted: false, env: "TWILIO_WHATSAPP_NUMBER" }`

The `env` property enables the dual credential model: per-org config in dashboard takes precedence, falls back to `.env` for self-hosted. The SDK's `resolveConfigWithEnv()` handles this automatically.

**Auth validation** (in `onValidateAuth`):

- Verify `accountSid` starts with `"AC"`
- Test Twilio API connection: `client.api.accounts(accountSid).fetch()`
- Follow pattern from `plugins/core/zendesk/src/index.ts:64-136`

**Route registration** (in `onInitialize`):

- `register.route("POST", "/webhook", webhookHandler)` — Twilio inbound
- `register.route("POST", "/deliver", deliverHandler)` — outbound delivery

### Modified Files

| File           | Change                                                                  |
| -------------- | ----------------------------------------------------------------------- |
| `.env.example` | Add `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_WHATSAPP_NUMBER` |

### Verify

- Plugin builds: `cd plugins/core/whatsapp && npm run build`
- Plugin discovered by plugin manager on server start
- Config renders in dashboard UI (auto-generated from schema)
- Validation rejects bad credentials, accepts good ones

---

## Phase 2: Inbound Messages (Twilio Webhook)

Receive WhatsApp messages, validate Twilio signatures, and create/reuse conversations.

### New Files

| File                                     | Purpose                            |
| ---------------------------------------- | ---------------------------------- |
| `plugins/core/whatsapp/src/webhook.ts`   | Twilio webhook handler             |
| `plugins/core/whatsapp/src/signature.ts` | `twilio.validateRequest()` wrapper |

### Webhook Handler Flow

1. Parse form-encoded body (Express already parses via `urlencoded` middleware in `server/main.ts:96`)
2. Validate Twilio signature (`X-Twilio-Signature` header) using `twilio.validateRequest(authToken, signature, originalUrl, params)`
3. Strip `whatsapp:` prefix from `From` field → E.164 phone number
4. Call plugin API `messages.receive` with `{ from: phone, content: Body, channel: "whatsapp", metadata: { profileName, waId, messageSid } }`
5. Return 200 (empty TwiML or JSON)

**Webhook URL** (configured in Twilio Console):

```
https://{domain}/v1/plugins/hay-plugin-whatsapp/webhook?organizationId={orgId}
```

This uses the plugin proxy at `server/routes/v1/plugins/proxy.ts` which forwards to the plugin worker.

### Critical Server-Side Fixes

**1. Fix `messages.receive` to reuse conversations** (`server/routes/v1/plugin-api/trpc.ts:84-92`)

The TODO at line 84 says "Find or create conversation by customer and channel". Currently it always creates new conversations. Fix:

```typescript
// Replace lines 84-92 with:
let conversation = await conversationRepository.findActiveByCustomerAndChannel(
  customer.id,
  channel,
  organizationId,
);

if (!conversation) {
  conversation = await conversationRepository.create({
    organization_id: organizationId,
    customer_id: customer.id,
    agent_id: agentId,
    channel,
    status: "open",
  });
}
```

Same fix needed in `messages.send` at line 180-188.

**2. Add `findActiveByCustomerAndChannel` to conversation repository** (`server/repositories/conversation.repository.ts`)

Query: find most recent conversation where `customer_id` matches, `channel` matches, `organization_id` matches, and `status` is in `["open", "processing", "pending-human", "human-took-over"]`.

**3. Fix proxy body forwarding** (`server/routes/v1/plugins/proxy.ts:73-77`)

The proxy always `JSON.stringify(req.body)` but doesn't update `Content-Type`. Since Express already parses the form body into an object, the JSON serialization works, but we should:

- Set `Content-Type: application/json` on the forwarded request (since we're sending JSON)
- Add `X-Original-Url` header so plugins can reconstruct the URL for signature validation
- Add `X-Original-Content-Type` header for reference

**4. Update customer metadata on subsequent messages** (`server/routes/v1/plugin-api/trpc.ts:59-72`)

When an existing customer is found, update their `external_metadata` with WhatsApp profile info (name, etc.) from the webhook payload.

### Verify

- Send test message from WhatsApp Sandbox → webhook fires → customer created → conversation created → orchestrator responds
- Second message reuses existing conversation (not creating a new one)
- Invalid Twilio signatures are rejected with 401
- Customer's `external_id` stores clean E.164 phone number

---

## Phase 3: Outbound Delivery (Channel Delivery Service)

The most architecturally novel piece. A server-side service that intercepts outbound bot messages on non-web channels and routes them to the plugin for delivery.

### Design

**Approach**: Subscribe to the same `websocket:events` Redis channel that the WebSocket service uses (see `server/services/websocket.service.ts:114`). When a `message_received` event arrives for a BOT_AGENT or HUMAN_AGENT message on a non-web channel, deliver it via the plugin.

**Why Redis subscriber (not hook in addMessage)**:

- Follows existing pattern (WebSocket service already does this)
- No modification to `conversation.entity.ts` or orchestrator
- Delivery failures don't block message creation
- Plugin-agnostic: works for any future channel

### New Files

| File                                          | Purpose                             |
| --------------------------------------------- | ----------------------------------- |
| `server/services/channel-delivery.service.ts` | Redis subscriber + delivery routing |

### Channel Delivery Service Logic

```
1. Subscribe to "websocket:events" Redis channel
2. On "message_received" event:
   a. Check payload.type is BOT_AGENT or HUMAN_AGENT → skip others
   b. Check payload.deliveryState === "sent" → skip "queued" (test mode)
   c. Look up conversation → check channel
   d. Skip if channel === "web"
   e. Find enabled plugin instance for this channel + org
   f. Get/start plugin worker
   g. POST to worker's /deliver endpoint with { to, content, messageId, conversationId }
   h. On success: update message.providerMessageId with Twilio SID
   i. On failure: log error (don't retry in v1)
```

**Plugin discovery**: Look up `PluginInstance` where `pluginId` matches a plugin with `category: "channel"` and the plugin's registered channel matches. For v1, use convention: plugin at `plugins/core/whatsapp` handles channel `"whatsapp"`. We can derive this from the plugin registry metadata or from the Source entity.

### Plugin-Side Deliver Route (`plugins/core/whatsapp/src/deliver.ts`)

```
1. Receive { to, content, messageId, conversationId }
2. Get Twilio credentials from plugin config
3. Call client.messages.create({ from: "whatsapp:{number}", to: "whatsapp:{to}", body: content })
4. Handle Twilio error 63016 (24h window expired) → return structured error
5. Return { success: true, providerMessageId: message.sid }
```

### Modified Files

| File                                        | Change                                                                     |
| ------------------------------------------- | -------------------------------------------------------------------------- |
| `server/main.ts`                            | Import and initialize `channelDeliveryService` after Redis init (~line 30) |
| `server/repositories/message.repository.ts` | Add `updateProviderMessageId(id, providerMessageId)` method                |

### Verify

- Send WhatsApp message → bot responds → response delivered to WhatsApp via Twilio
- Test mode: bot response stays queued, not delivered until approved
- Non-WhatsApp conversations (web) unaffected
- Twilio errors logged, don't crash the service
- `providerMessageId` (Twilio SID) stored on message

---

## Phase 4: Template Management (Deferred)

> Template management is not needed for v1. Since we close conversations when the 24h window expires, the AI never needs to send templates. This phase can be implemented later when we want to support business-initiated outreach.

### 24h Window Handling (Implemented in Phase 3)

In the deliver handler, when Twilio returns error code 63016 (session expired):

1. Return `{ success: false, error: "24h_window_expired" }`
2. Channel delivery service logs the error
3. Conversation will be closed by existing inactivity timeout
4. Customer's next message opens a new conversation

---

## Phase 5: Dashboard UI

### New Files

| File                                                          | Purpose                         |
| ------------------------------------------------------------- | ------------------------------- |
| `plugins/core/whatsapp/components/settings/AfterSettings.vue` | Setup guide                     |
| `plugins/core/whatsapp/components/index.ts`                   | Component exports               |
| `plugins/core/whatsapp/vite.config.ui.ts`                     | UI build config (match Zendesk) |

### Setup Guide Content

Step-by-step guide registered as `after-settings` UI page (follow `plugins/core/zendesk/components/settings/AfterSettings.vue` pattern):

1. Create/log into Twilio account
2. Enable WhatsApp Sandbox (dev) or register number (prod)
3. Copy Account SID + Auth Token from Twilio Console
4. Configure webhook URL in Twilio: `https://{domain}/v1/plugins/hay-plugin-whatsapp/webhook?organizationId={orgId}`
5. Enter WhatsApp number in plugin settings

The webhook URL should be dynamically generated using the current domain and org ID.

### Verify

- Setup guide renders in plugin settings page
- Webhook URL is correct and copyable
- Plugin config fields show env fallback status correctly

---

## Risk Areas

| Risk                                               | Severity | Mitigation                                                         |
| -------------------------------------------------- | -------- | ------------------------------------------------------------------ |
| Proxy body encoding mismatch                       | HIGH     | Fix in Phase 2 — set Content-Type and forward original URL         |
| `messages.receive` creates duplicate conversations | HIGH     | Fix in Phase 2 — add `findActiveByCustomerAndChannel`              |
| Twilio signature URL reconstruction                | MEDIUM   | Pass original URL via `X-Original-Url` header from proxy           |
| 24h window failures                                | LOW      | Return structured error, let inactivity timeout close conversation |
| Redis subscription ordering                        | LOW      | Channel delivery subscribes same as WebSocket — proven pattern     |

---

## No Database Migrations Needed

Existing schema fully supports WhatsApp:

- `Conversation.channel` enum: already has `"whatsapp"`
- `Customer.external_id`: stores E.164 phone
- `Customer.external_metadata`: stores `{ whatsapp: { profileName, waId } }`
- `Message.providerMessageId`: stores Twilio SID
- `Organization.settings.channelAgents`: maps `"whatsapp"` to agent ID

Only new query: `findActiveByCustomerAndChannel` (no schema change).

---

## Implementation Order

```
Phase 1 (skeleton) → Phase 2 (inbound) → Phase 3 (outbound) → Phase 5 (UI)
```

Each phase is independently verifiable. Phase 5 can start in parallel with Phase 3.

---

## All Files Summary

### New Files

- `plugins/core/whatsapp/package.json`
- `plugins/core/whatsapp/tsconfig.json`
- `plugins/core/whatsapp/src/index.ts`
- `plugins/core/whatsapp/src/webhook.ts`
- `plugins/core/whatsapp/src/deliver.ts`
- `plugins/core/whatsapp/src/signature.ts`
- `plugins/core/whatsapp/components/settings/AfterSettings.vue`
- `plugins/core/whatsapp/components/index.ts`
- `plugins/core/whatsapp/vite.config.ui.ts`
- `server/services/channel-delivery.service.ts`

### Modified Files

- `.env.example` — Twilio env vars
- `server/main.ts` — Initialize channel delivery service
- `server/routes/v1/plugins/proxy.ts` — Fix body encoding + add original URL header
- `server/routes/v1/plugin-api/trpc.ts` — Fix messages.receive/send to reuse conversations
- `server/repositories/conversation.repository.ts` — Add `findActiveByCustomerAndChannel()`
- `server/repositories/message.repository.ts` — Add `updateProviderMessageId()`

---

## Key Reference Files

- **Zendesk plugin** (pattern to follow): `plugins/core/zendesk/src/index.ts`, `package.json`, `tsconfig.json`, `vite.config.ui.ts`, `components/`
- **Plugin SDK types**: `packages/plugin-sdk/types/` (plugin.ts, hooks.ts, contexts.ts, register.ts, mcp.ts)
- **Plugin proxy**: `server/routes/v1/plugins/proxy.ts`
- **Plugin API**: `server/routes/v1/plugin-api/trpc.ts` (messages.receive, messages.send, getAgentForChannel)
- **WebSocket Redis subscriber**: `server/services/websocket.service.ts:114` (pattern for channel delivery)
- **Conversation entity**: `server/database/entities/conversation.entity.ts` (channel enum, addMessage, cooldown)
- **Customer entity**: `server/database/entities/customer.entity.ts` (external_id, external_metadata)
- **Message entity**: `server/database/entities/message.entity.ts` (providerMessageId, deliveryState)
- **Organization settings**: `server/types/organization-settings.types.ts:308` (channelAgents)
- **Server startup**: `server/main.ts` (initialization order, Redis, WebSocket, plugins)
