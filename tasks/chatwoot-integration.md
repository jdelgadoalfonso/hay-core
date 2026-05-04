# Chatwoot Channel Integration — Plan

## Goal

Integrate Chatwoot as a first-class channel plugin in Hay so that Chatwoot-hosted conversations can be handled by Hay's AI orchestrator, with bi-directional message flow, human-agent propagation back into Hay, and clean escalation from AI → Chatwoot humans.

## Acceptance Criteria

- A Chatwoot Agent Bot can be connected to a Hay organization via plugin config (baseUrl, accountId, botAccessToken, webhookSecret).
- Incoming Chatwoot messages (`message_created`, `message_type=0`) trigger the Hay orchestrator and produce AI responses sent back via Chatwoot's API.
- Bot's own outgoing messages, private internal notes, and template messages are filtered out of the ingest path.
- When the orchestrator emits HANDOFF, the Chatwoot conversation is moved to `status=open` (out of bot's `pending`), optionally assigned to a configured team, and an internal note with handoff reason is posted.
- When a Chatwoot human agent replies, the message is propagated back to Hay as a `HUMAN_AGENT` message and the conversation flips to `human-took-over` so the orchestrator stops running.
- Both Chatwoot Cloud and self-hosted instances work without code differences.
- No hardcoded plugin IDs in Hay core. No new channel-aware code paths outside the plugin.

## Architectural Decisions (locked)

1. Escalation is dispatched via a new plugin endpoint `POST /escalate`, called by `channel-delivery.service` when `conversation_status_changed` → `pending-human`.
2. `Conversation.channel` converts from Postgres enum → `varchar(64)`. Future channel plugins won't need migrations.
3. Both Chatwoot Cloud and self-hosted are first-class. Config asks for `baseUrl`.
4. v1 correlation: one active Hay conversation per (customer, channel) — same as WhatsApp. Chatwoot `conversation_id` stored in `Conversation.metadata.chatwoot.conversationId` for outbound lookup.
5. Human-agent replies from Chatwoot propagate to Hay as `HUMAN_AGENT` messages via an extended `messages.receive` that accepts `senderType`.

## Deferred (v2)

- Multiple concurrent Chatwoot conversations per contact (Option A correlation by external conversation ID).
- Release-to-AI from Hay dashboard flipping the Chatwoot conversation state back.
- Auto-provisioning Chatwoot agent bots via Platform API for self-hosted customers.

## Slices

### Slice A — channel enum → varchar

- Migration: `ALTER TABLE conversations ALTER COLUMN channel TYPE varchar(64) USING channel::text; DROP TYPE conversations_channel_enum;`
- Entity: `conversation.entity.ts` — column becomes `varchar(64)`, type becomes `string`.
- Loosen zod schemas at `plugin-api/trpc.ts:54` and `:171` to `z.string().min(1).max(64)`.

### Slice B — messages.receive extension

- Add optional `senderType: "customer" | "human_agent"` (default `"customer"`).
- Add optional `externalConversationId: string` — merged into `conversation.metadata[channel].conversationId`.
- Add optional `externalMetadata: Record<string, unknown>` — merged into `conversation.metadata[channel]`.
- When `senderType === "human_agent"`: message type is `HUMAN_AGENT`, conversation status flips to `"human-took-over"` (via `conversation.assignToUser` or direct update — follow existing takeover path).

### Slice C — channel-delivery.service

- `/deliver` payload extended with `conversationMetadata: conversation.metadata` so plugins can read external IDs.
- New handler: subscribe to `conversation_status_changed` events, on `status === "pending-human"` call plugin `/escalate` with `{ conversationId, conversationMetadata, reason, handoffMessage }`.
- `/escalate` is optional for plugins — 404 is silently tolerated so WhatsApp etc. don't have to implement it.

### Slice D — plugin scaffolding

- `plugins/core/chatwoot/package.json`, `tsconfig.json`, `vite.config.ui.ts`, dir structure mirroring `plugins/core/whatsapp/`.

### Slice E — Chatwoot API client

- `src/chatwoot-api.ts`: `sendMessage`, `toggleStatus`, `assignTeam`, `postPrivateNote`. Header: `api_access_token`. Handles both `app.chatwoot.com` and self-hosted base URLs.

### Slice F — handlers

- `src/webhook.ts`: verify shared-secret URL token, filter out bot echoes / private / template messages, detect human-agent replies vs customer messages, call `plugin-api` `messages.receive` with correct `senderType`.
- `src/deliver.ts`: read `chatwootConversationId` from `conversationMetadata`, POST to Chatwoot messages API, return `{success, providerMessageId}`.
- `src/escalate.ts`: `toggleStatus("open")` + optional `assignTeam` + optional private note with handoff reason.
- `src/plugin-api.ts`: copy WhatsApp pattern.

### Slice G — src/index.ts

- `register.config`: baseUrl, accountId, botAccessToken (encrypted), webhookSecret (encrypted), defaultEscalationTeamId (optional).
- `register.auth.apiKey` using botAccessToken.
- `register.route` for `/messages`, `/deliver`, `/escalate`.
- `onValidateAuth`: call `GET /api/v1/accounts/{accountId}/profile` with bot token to validate.
- `onStart`: construct API client with baseUrl + token, store in closure.

### Slice H — onboarding UI

- `components/settings/AfterSettings.vue`: step-by-step "Create an Agent Bot → paste credentials → copy this webhook URL into your Chatwoot bot's outgoing_url → assign bot to your inbox".

### Slice I — verification

- `npm run typecheck` clean.
- Commit to `claude/chatwoot-api-integration-3upgq`, push.

## Risks & Notes

- Enum→varchar migration: must also rewrite the original consolidated schema migration's down path or leave it alone (forward-only is the pragmatic answer — see `1758718343166-ConsolidatedInitialSchema.ts`).
- Webhook URL shared-secret: path-embedded secret (`/plugins/webhooks/chatwoot/messages/{secret}`) won't work with the existing generic webhook receiver pattern. Instead the secret lives in a query param (`?secret=...`) checked inside the plugin's own handler. Documented in onboarding.
- Bot echo filtering: Chatwoot fires `message_created` for outgoing bot messages. Filter by `sender.type === "AgentBot"` and the `account_id/bot_id` match.
- 429 handling on Chatwoot API: exponential backoff in the thin client.
- Token at rest: bot access token and webhook secret are stored as `encrypted: true` config fields (handled by the plugin SDK).

## Out of Scope for this PR

- End-to-end live test against real Chatwoot instance (user will do this after push).
- Auto-assigning the bot to inboxes (requires user token, not bot token).
- Playbook/agent configuration UI changes beyond the plugin's own settings.
