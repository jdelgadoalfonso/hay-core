# Websites as Connected Sources (Option B — built-in importer)

## Goal / Acceptance Criteria

- A user can connect a website URL and it appears in `/documents/sources` like Confluence:
  display name, sync status pill, "last synced", document count.
- Pages sync as `Document`s tied to a `DocumentSource` (sourceType `website`), with
  `externalUpdatedAt` populated from sitemap `<lastmod>` / `Last-Modified` so the UI
  shows when each page was last updated, and incremental sync is cheap.
- Re-sync (manual + scheduled), incremental delta, and weekly full-sweep deletion
  reconciliation all work via the existing sync engine — no engine special-casing
  beyond importer resolution.
- The old one-shot web import (discoverWebPages/importFromWeb/recrawl) is REMOVED;
  the import.vue website step creates a website source instead.

## Decisions (confirmed with user)

- No-sitemap sites: sitemap-first; bounded single crawl fallback (maxPages cap),
  no per-page lastmod there. Log the limitation.
- Converge now: remove the one-shot web import endpoints in this PR.
- Built-in importer uses sentinel `pluginId = "core:website"`, `sourceType = "website"`.

## Plan

- [ ] Backend: `web-scraper.service.ts` — extract sitemap `<lastmod>` + `Last-Modified` header
- [ ] Backend: new `WebsiteImporter` implementing DocumentImporterContract over web-scraper + HtmlProcessor
- [ ] Backend: `document-source-sync.service.ts` — `resolveImporter(source)` pluggable hook
- [ ] Backend: `document-sources/index.ts` — `createWebsite({ url })` mutation (SSRF-validated)
- [ ] Frontend: import.vue website step → `createWebsite` → redirect to sources/[id]
- [ ] Frontend: `getSourceIcon` Globe for `website`; i18n labels
- [ ] Remove: discoverWebPages/importFromWeb/recrawl endpoints + dead web-import code paths
- [ ] Verify: typecheck, server tests (add WebsiteImporter unit + resolver test), manual repro

## Working Notes

- Sync engine importer resolution: document-source-sync.service.ts:159 + createImporter (516).
- createImporter takes (router, source) and returns DocumentImporterContract — built-in
  importer is constructed WITH source, ignores instanceId, reads base URL from source.config.url.
- Contract: server/types/plugin-sdk.types.ts (DocumentImporterContract + page/root/change types).
- Sources UI already generic; thumbnail <img> 404s for core:website and falls back to getSourceIcon.
- discover() must be resumable via offset cursor — re-fetch sitemap each page, slice by offset.

## Results

**What changed**

- Websites are now first-class `DocumentSource`s (sourceType `website`, sentinel pluginId
  `core:website`) backed by a built-in importer — no plugin required. They show in
  /documents/sources with sync status, "last synced", doc count, and per-page `externalUpdatedAt`
  from sitemap `<lastmod>` / `Last-Modified` (the "when was it updated" data the user wanted).
  Re-sync, incremental delta, and weekly full-sweep deletion reconciliation all work via the
  existing sync engine, unchanged.
- New: server/services/importers/website-importer.ts; createWebsite mutation; resolveImporter() seam.
- Converged: the old one-shot web import is gone; the import.vue "website" step connects a source.

**Verification**

- server typecheck ✅ · server lint (changed files) ✅
- dashboard typecheck ✅ (API_DOMAIN=… NODE_ENV=development) · dashboard lint (changed files) ✅
- jest website-importer.test.ts ✅ 5/5 · jest document-source-sync.test.ts ✅ (plugin path intact)

**Live sync progress (added)**

- Sync engine streams throttled progress onto the running job's `data.progress`
  ({ phase, total, discovered, processed, created, updated, currentTitle, currentUrl }).
- WebsiteImporter.discover reports `total` (enumerated count) → "X of Y" bar.
- Source detail page shows a live "Discovering / Importing pages…" card (counts + current page),
  polls every 2s while a sync job is active (queued/processing/running), and reflects on connect.
- Verified: server typecheck/lint ✅, dashboard typecheck/lint ✅, importer + sync tests ✅.

**Bugfix: duplicate sync jobs / frozen progress (regression)**

- Symptom: connecting one website produced a new "Sync …" job every 60s, all stuck
  "processing", 0 documents, and the live card frozen at "Importing pages… Starting…".
- Root cause: the 60s `document-source-sync-dispatcher` enqueues every source
  `findDueForSync` returns. A brand-new source has `last_synced_at = NULL` and
  `last_sync_status = NULL` (the 'running' flag is only set once a worker picks the
  job up via `markRunning`), so it matched "due" every tick → one new job per minute.
  The UI polls the newest active job, but each duplicate bailed at `markRunning`
  ("already running") and never reported progress → frozen "Starting…", while the
  oldest job silently did the work.
- Fix: `enqueueSync` is now idempotent — `jobRepository.findActiveSyncJob(sourceId)`
  returns any pending/queued/processing sync job for the source and `enqueueSync`
  reuses it instead of creating a duplicate. Single chokepoint, covers both the
  dispatcher and the manual "Sync now" / connect paths. Collapsing to one job also
  restores live progress (the polled job is now the one doing the work).
- Regression test: `enqueueSync is idempotent` in the sync integration suite.
- Verified: server typecheck ✅ · lint ✅ · jest document-source-sync ✅ 7 passed / 2 skipped.
- Note: existing piled-up jobs drain on their own (each reaches terminal via
  completeJob); the fix takes effect after the server restarts to load new code.

**Known follow-ups (optional)**

- Unused i18n keys remain (documents.import.discovery._, .metadata._, processing web keys,
  webUrl.discoverPages, steps.selectPages/addMetadata/processing) — harmless; prune later.
- No-sitemap crawl-fallback ordering is best-effort across partial-run resumes (sitemap path is
  deterministic). Acceptable for v1; weekly full sweep reconciles.
