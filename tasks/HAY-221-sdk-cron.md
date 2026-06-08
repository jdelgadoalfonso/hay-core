# HAY-221 — SDK Cron Jobs + Shopify Token Refresh

**Epic:** HAY-158 Shopify Marketplace · **Track:** B (self-hosted) · independent of 218/219/220.

## Goal / Acceptance Criteria

SDK cron infra:

- [ ] Plugins register cron jobs via SDK (`register.cron({ name, schedule, handler, retryPolicy })`)
- [ ] Scheduler respects worker isolation (one worker per org per plugin)
- [ ] Jobs run in org-plugin context with access to credentials + config
- [ ] Jobs can update stored credentials (token-refresh use case) — `ctx.auth.update()`
- [ ] Error handling with configurable retry policy
- [ ] Job registration/deregistration on plugin enable/disable per org

Shopify plugin:

- [ ] Registers cron job for token refresh (every 20h, buffer before 24h expiry)
- [ ] Job performs Shopify client-credentials grant for fresh access token
- [ ] Job updates stored access token for the org
- [ ] Graceful handling if refresh fails (retry, alert, no stale-token use)
- [ ] Self-hosted docs updated with Dev Dashboard setup

## Key constraint

Plugins are out-of-process HTTP workers, spawned on demand, idle-killed after 5 min.
Auth runtime API is read-only today → must add a credential-update callback to core.

## Plan (thin vertical slices)

### Slice 1 — SDK surface

- [ ] `register.cron()` type + impl in PluginRegistry (`packages/plugin-sdk`)
- [ ] `HayCronContext` type: org, config.get(), auth.get(), auth.update(creds), logger
- [ ] Expose declared crons in `/metadata`
- [ ] Worker `POST /cron/:name` endpoint → builds context, runs handler
- [ ] `auth.update()` → authenticated callback to core

### Slice 2 — Core bridge

- [ ] `PluginCronService`: read declared crons, register per enabled org in `SchedulerService`
- [ ] Register on enable / unregister on disable
- [ ] On fire: ensure worker spawned → POST /cron/:name
- [ ] Core callback endpoint → `pluginInstanceRepository.updateAuthState()` + re-inject env

### Slice 3 — Shopify plugin (first consumer)

- [ ] `plugins/core/shopify` scaffold (mirror twenty archetype)
- [ ] Config: clientId, clientSecret, shopDomain
- [ ] `refresh_shopify_token` cron handler (client-credentials grant → auth.update)
- [ ] Docs: self-hosted Dev Dashboard setup

### Verify

- [ ] Typecheck (server + sdk), lint
- [ ] Unit test: cron handler refresh logic + retry
- [ ] Manual: register a 1-min test cron, confirm fire + credential update persists

## Working notes

- Reuse `SchedulerService` (node-cron, retries, history) — don't build a new scheduler.
- `pluginInstanceRepository.updateAuthState(instanceId, orgId, authState)` already exists.
- Existing core `oauth-token-refresh.job` uses refresh-token grant; Shopify uses client-credentials grant — genuinely different, justifies plugin-side handler.
- After refresh, running MCP still holds old token in env → must trigger worker re-init.

## Results (implemented)

**SDK** (`packages/plugin-sdk`)

- `types/cron.ts` (new): `CronJobOptions`, `HayCronContext`, `HayCronAuthAPI` (adds `update()`), `CronJobDescriptor`.
- `register.cron()` added to `HayRegisterAPI` + impl/validation in `sdk/register.ts`; storage in `sdk/registry.ts`.
- `sdk/cron-runtime.ts` (new): cron auth API that buffers `update()` for return to core.
- `runner/http-server.ts`: `crons` added to `/metadata`; new `POST /cron/:name` runs the handler in-worker and returns any staged credential update.

**Core** (`server`)

- `types/plugin-sdk.types.ts`: `crons?` added to `PluginMetadata`.
- `services/plugin-cron.service.ts` (new): registers one SchedulerService job per (org, plugin, cron); on fire wakes worker → POST /cron/:name → persists `updateAuthState()` + restarts worker.
- `main.ts`: `pluginCronService.initialize()` after plugin manager boot.
- `routes/v1/plugins/plugins.handler.ts`: register on enable / unregister on disable.

**Shopify plugin** (`plugins/core/shopify`, new): config (shopDomain/clientId/clientSecret/apiVersion), client-credentials grant, `refresh_shopify_token` cron (`0 */20 * * *`), minimal MCP (`shopify_get_shop`; full tools = HAY-219). README with self-hosted Dev Dashboard setup.

**Verification**

- `plugin-sdk` typecheck + build: pass. `server` typecheck: pass. Shopify plugin `tsc` build: pass.
- SDK tests: 4 new cron tests pass. Pre-existing 5 failures in `register.test.ts` "UI Extension Registration" are unrelated (legacy `register.ui()` form, not touched).
- Not yet done: live end-to-end run against a real Shopify dev store (needs HAY-220 credentials); HAY-219 MCP tool surface.
