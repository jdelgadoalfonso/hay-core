# Per-channel agent routing (many-to-many)

## Goal / acceptance criteria

- An agent can be assigned to multiple channels; a channel can list multiple agents (M2M).
- At runtime each incoming message still resolves to exactly ONE agent for its channel.
- Agent settings page shows a "Channels" card listing the org's enabled channels (incl. built-in Web Chat) as toggles.
- The Channels card is hidden when the org has zero enabled channels.
- `channelAgents` org-setting scaffolding removed (alpha, no back-compat).

## Data model decision

- Add `channels: string[]` (text[] , default `{}`) to Agent. This IS many-to-many: a channel id can appear on many agents, an agent can have many channels.
- Channel ids are the canonical `manifest.channel` strings (e.g. "instagram") + built-in "web".

## Runtime resolution (temporary rule, documented as TODO)

For a given channel, `getAgentForChannel`:

1. Agents whose `channels` include the channel -> if multiple, prefer org default agent if among them, else deterministic (earliest created_at).
2. Else org `defaultAgentId`.
3. Else first agent.

## Tasks

- [ ] Agent entity: add `channels` text[] column
- [ ] Migration: add `channels` to agent
- [ ] Remove `channelAgents` from organization-settings.types.ts (+ doc migration ref)
- [ ] Rewrite `getAgentForChannel` (plugin-api/trpc.ts) to use agent.channels
- [ ] agents create/update routes: accept `channels: string[]`
- [ ] New tRPC procedure: list org enabled channels (id, name, thumbnail) incl. Web Chat
- [ ] Dashboard: Channels card on agents/[id].vue bound to agent.channels; hidden when no channels
- [ ] Regenerate tRPC types
- [ ] Verify: typecheck server + dashboard; manual routing check

## Results

- Agent entity: added `channels text[]` (migration 1782000000004, applied).
- Removed `channelAgents` from organization-settings types; rewrote `getAgentForChannel`
  to resolve via `agent.channels` (assigned → prefer default → earliest; else org default; else first).
- agents create/update accept `channels: string[]`; service passes through.
- `plugins.getAll` now returns `channel` (manifest.channel) per plugin.
- agents/[id].vue: "Channels" card (switches for Web Chat + enabled channel plugins),
  hidden when org has zero channels; bound to `form.channels`.
- Verified: server typecheck clean; dashboard typecheck clean for agents page
  (4 pre-existing unrelated errors in conversations/customers remain). Migration applied OK.

## Working notes

- channelAgents was unused scaffolding (types + getAgentForChannel + doc-only migration 1764863000000).
- Enabled channel plugins: filter plugins by `enabled && type.includes("channel")`; channel id from manifest.channel (not currently surfaced by getAll - must add).
- Web Chat / "web" is always-on built-in (no plugin instance).
