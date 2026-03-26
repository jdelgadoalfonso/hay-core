# Git Plugin Sync — Implementation Plan

## Goal

Allow organizations to install and sync custom plugins from Git repositories (GitHub first, provider-agnostic design). This is a **core service**, not a plugin.

## Key Decisions

- **GitHub App OAuth only** — no PATs. Self-hosted users create their own GitHub App.
- **Provider-agnostic interface** — `GitProvider` abstraction, `GitHubProvider` first, GitLab/Bitbucket later.
- **Core service** — not MCP, not a plugin. Direct backend service used by the system.
- **Multi-server sync** — reuses existing pattern: store archive in S3, auto-restore on startup.

---

## Architecture

### New Entity: `git_connections`

```
git_connections
├── id (uuid, PK)
├── organizationId (FK → organization)
├── provider ("github" | "gitlab" | "bitbucket" | "custom")
├── label (user-friendly name, e.g. "Our GitHub Org")
├── baseUrl (API base URL — handles GHE/self-hosted GitLab)
├── credentials (JSONB, encrypted)
│   ├── For GitHub App: { appId, privateKey, installationId }
│   ├── For GitLab OAuth: { accessToken, refreshToken, expiresAt }
│   └── (provider-specific, opaque to the generic layer)
├── status ("connected" | "disconnected" | "error")
├── connectedAt (timestamptz)
├── lastUsedAt (timestamptz)
├── createdAt / updatedAt
```

### Extended `plugin_registry` columns

```
sourceType: "core" | "custom" | "git"    -- add "git"
gitConnectionId: uuid (FK → git_connections, nullable)
gitRepo: varchar  -- "owner/repo"
gitRef: varchar   -- branch or tag to track
gitCommitSha: varchar  -- currently installed commit
gitLastSyncAt: timestamptz
gitSyncError: text
```

### Provider Interface

```typescript
interface GitProvider {
  id: string; // "github" | "gitlab" | ...

  // Auth
  getAuthUrl(connection: GitConnection, callbackUrl: string): Promise<string>;
  handleCallback(connection: GitConnection, code: string): Promise<GitCredentials>;
  refreshCredentials(connection: GitConnection): Promise<GitCredentials>;
  validateConnection(connection: GitConnection): Promise<boolean>;

  // Repo operations
  listRepos(connection: GitConnection): Promise<GitRepo[]>;
  getLatestCommit(connection: GitConnection, repo: string, ref: string): Promise<string>; // returns SHA
  downloadArchive(connection: GitConnection, repo: string, ref: string): Promise<Buffer>; // tarball/zip
}
```

### GitConnectionService (orchestrator)

```typescript
class GitConnectionService {
  private providers: Map<string, GitProvider>;

  // Connection management
  createConnection(orgId, provider, label, baseUrl?): Promise<GitConnection>;
  initiateAuth(connectionId): Promise<{ authUrl: string }>;
  handleAuthCallback(provider, code, state): Promise<GitConnection>;
  deleteConnection(connectionId): Promise<void>;

  // Plugin operations
  installPluginFromRepo(connectionId, repo, ref): Promise<PluginRegistry>;
  syncPlugin(pluginId): Promise<{ updated: boolean; newSha?: string }>;
  syncAllPlugins(orgId): Promise<SyncReport>;
}
```

### Flow: Install Plugin from GitHub

1. User has a `git_connection` with GitHub (already authenticated)
2. User selects a repo from `listRepos()` (or pastes owner/repo)
3. Backend calls `downloadArchive(connection, repo, ref)` → gets tarball
4. Validates archive (same checks as ZIP upload — package.json, hay-plugin field)
5. Stores archive in S3 (same path pattern as ZIP uploads)
6. Extracts to `plugins/custom/{orgId}/{pluginId}/`
7. Registers in `plugin_registry` with `sourceType: "git"` + git metadata
8. Runs install + build

### Flow: Sync/Update

1. Scheduler job runs every N minutes (configurable, default 15min)
2. For each `sourceType: "git"` plugin, calls `getLatestCommit()`
3. Compares with stored `gitCommitSha`
4. If different: download new archive → store in S3 → extract → re-register → rebuild
5. Stop running workers before extraction, restart after
6. Update `gitCommitSha` and `gitLastSyncAt`

### GitHub App Specifics (GitHubProvider)

- GitHub Apps use JWT + installation tokens (not standard OAuth2 refresh)
- Flow:
  1. User visits GitHub App installation URL
  2. GitHub redirects back with `installation_id` and `setup_action`
  3. We store `installation_id` in `git_connections.credentials`
  4. To make API calls: sign JWT with app private key → exchange for installation token (1hr TTL)
  5. Generate fresh installation token for each operation (they're cheap and short-lived)
- Env vars needed: `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, `GITHUB_APP_WEBHOOK_SECRET` (optional)
- Self-hosted: users set these env vars with their own GitHub App's values

### Multi-Server Sync

Already handled by existing patterns:

- Archive stored in S3 → any server can restore
- `plugin_registry` in PostgreSQL → shared state
- On startup, `pluginManagerService` checks for missing directories and restores from S3
- Could optionally use Redis pub/sub to notify other servers of sync events (optimization, not required)

---

## Implementation Order

### Phase 1: Foundation

- [ ] Create `git_connections` entity + migration
- [ ] Add git columns to `plugin_registry` + migration
- [ ] Define `GitProvider` interface
- [ ] Create `GitConnectionService`
- [ ] Create `GitHubProvider` (GitHub App auth + repo operations)

### Phase 2: API

- [ ] tRPC routes: git connections CRUD
- [ ] tRPC routes: list repos, install plugin from repo
- [ ] tRPC routes: manual sync trigger
- [ ] OAuth callback handler for GitHub App installation

### Phase 3: Automation

- [ ] Scheduled sync job (poll for updates)
- [ ] Redis pub/sub notification for multi-server (optional)

### Phase 4: Dashboard UI

- [ ] Git connections settings page
- [ ] GitHub App setup wizard (for self-hosted)
- [ ] Repo browser / plugin installer
- [ ] Sync status and logs

### Future

- [ ] GitLab provider
- [ ] Bitbucket provider
- [ ] Webhook-based sync (instead of polling)
- [ ] Document sync from Git repos (reuses same connections)
