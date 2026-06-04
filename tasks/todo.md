# Build: Generic Twenty CRM Plugin

## Goal

A generic Hay plugin (`hay-plugin-twenty`) connecting **any** Twenty CRM workspace
(Twenty Cloud or self-hosted) via the user's own base URL + API key. Inspired by the
workspace-specific backoffice MCP at `hay-website/hay-backoffice/src`, but stripped of all
custom fields (source/stage/vertical/fit/signals/COMPANY_SIZE) so it works for anyone.

Archetype **A** (local bundled MCP over stdio), gold-standard reference: `klaviyo`.

## Acceptance criteria

- [ ] `package.json` has `hay-plugin` block, ESM, no `manifest.json`.
- [ ] `src/index.ts` default-exports `defineHayPlugin`; config = baseUrl + apiKey (encrypted).
- [ ] `onValidateAuth` does a real round-trip; `onStart` gates on creds; `onDisable` present.
- [ ] `mcp/` plain-JS server using native `fetch` (no axios), `@modelcontextprotocol/sdk` + zod.
- [ ] Tools cover standard objects (people/companies/notes/tasks) + **generic record CRUD** +
      **metadata** (so custom objects/fields in the user's own workspace are reachable).
- [ ] No workspace-specific fields hardcoded. No secrets. `strict: true`.
- [ ] `npm run build` passes; tools list cleanly.

## Plan

- [x] Read SDK contract, archetypes, anti-patterns
- [x] Study backoffice Twenty service + MCP tools (API shapes)
- [x] Study klaviyo (gold standard) scaffolding
- [x] Create worktree `plugin/twenty-crm`
- [ ] Scaffold package.json / tsconfig / src/index.ts / i18n / README
- [ ] mcp/ : lib/client.js (fetch+retry+error fmt), lib/format.js (ok/fail/blocknote)
- [ ] mcp/tools: people, companies, notes, tasks, metadata, records (generic)
- [ ] mcp/index.js boot + register all tools
- [ ] Build verification

## Working notes

- Twenty REST base: `${baseUrl}/rest`; metadata: `${baseUrl}/rest/metadata`.
- List response: `{ data: { <plural>: [...] }, pageInfo: { hasNextPage, endCursor } }`.
- Single: `{ data: { <singular>: {...} } }`. Create: `{ data: { create<Singular>: {...} } }`.
- Filter: `field[eq]:value`, `and(...)`, `or(...)`, `like:%x%`. Pagination: `starting_after` cursor.
- Notes/tasks bodies use `bodyV2: { blocknote, markdown }`. Attach via `/noteTargets` `/taskTargets`.
- Generic tools key off `objectNamePlural` in the path; unwrap first key of `data`.
- build-plugins.sh installs `mcp/` deps at build time → node_modules stays gitignored.

## Results

**Built** `plugins/core/twenty/` — generic Twenty CRM plugin (archetype A, klaviyo pattern).

Files:

- `package.json` (hay-plugin block, ESM, mcp+auth), `tsconfig.json` (strict), `README.md`, `i18n/en.json`, `thumbnail.jpg` (placeholder — replace with Twenty logo).
- `src/index.ts` — `defineHayPlugin`; config `baseUrl` + `apiKey` (encrypted); `onValidateAuth` round-trips `/rest/metadata/objects`; `onStart` gates on creds + spawns stdio MCP; `onConfigUpdate`/`onDisable` present (no `onEnable`).
- `mcp/index.js` boot + `mcp/lib/{client,format}.js` (native fetch, retry on 429/5xx, error-body surfacing, blocknote/link/filter helpers).
- `mcp/tools/{people,companies,notes,tasks,metadata,records}.js` — **26 tools**, all `twenty_`-prefixed.

Generic design: `twenty_list_objects` + `twenty_get_select_options` (schema discovery) and
`twenty_{list,get,create,update,delete}_record` reach ANY object incl. custom ones. Typed
people/company/note/task tools are the friendly fast-path. No workspace-specific fields hardcoded.

Verification:

- `npm run build` (tsc) → exit 0, emits `dist/index.js`.
- MCP smoke test: all 26 tools register; `normalizeBaseUrl` strips `/rest`.
- Full stdio handshake (initialize + tools/list via SDK Client) → 26 tools with real input schemas.
- Git hygiene: dist + node_modules + mcp/node_modules ignored; only source staged; no secrets.
- `ilike` (case-insensitive) confirmed supported by Twenty REST API (docs).

Assumptions / follow-ups:

- thumbnail.jpg is a placeholder copied from klaviyo — swap for Twenty's logo.
- Not yet exercised against a live Twenty workspace (no test creds); request shapes mirror the proven backoffice integration.
