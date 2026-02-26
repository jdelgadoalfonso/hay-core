# Lessons Learned

## 2026-02-26: Security Review (HAY-80 branch)

### Inverted condition bug in sandbox env var blocklist
- **Failure mode**: `if (!baseEnv[varName]) delete sandboxEnv[varName]` — deletes only vars that DON'T exist (no-op), keeps dangerous vars
- **Detection signal**: Code review / security audit
- **Prevention**: Always test sandbox isolation with actual dangerous env vars present

### Pino redact `*` only covers 1 nesting level
- **Failure mode**: `*.password` catches `{a: {password}}` but NOT `{a: {b: {password}}}`
- **Prevention**: Generate paths at multiple depths (`field`, `*.field`, `*.*.field`)

### JWT access/refresh token confusion
- **Failure mode**: Same secret + no `type` claim = tokens are interchangeable
- **Prevention**: Always use separate signing secrets AND a `type` discriminator claim for different token types

### Timing-unsafe string comparison on HMAC signatures
- **Failure mode**: `===` leaks timing information on signature verification
- **Prevention**: Always use `crypto.timingSafeEqual()` for any secret/signature comparison

### Plugin manifest can request arbitrary host env vars
- **Failure mode**: Plugin `env` field in manifest could request `JWT_SECRET`, `DB_PASSWORD`, etc.
- **Prevention**: Maintain a deny-list of sensitive env vars that plugins can never access

### Password reset token O(n) argon2 scan
- **Failure mode**: `verifyResetToken` loads ALL users with pending resets and runs argon2 verify on each — O(n) expensive operations, potential DoS
- **Prevention**: Use SHA-256 for high-entropy tokens (256-bit random) instead of argon2. SHA-256 enables direct DB lookup. Argon2 is only needed for low-entropy secrets (passwords)

### optionalAuth swallows authorization errors
- **Failure mode**: `catch {}` suppresses ALL errors including FORBIDDEN, so a user with invalid org access silently becomes unauthenticated instead of getting an error
- **Prevention**: Only catch authentication failures; re-throw authorization errors (FORBIDDEN)

### Dynamic column names in SQL queries
- **Failure mode**: `entity.${userInput}` in TypeORM query builder allows SQL injection through filter/sort/search field names
- **Prevention**: Validate all dynamic identifiers against `/^[a-zA-Z_][a-zA-Z0-9_]*$/` before interpolation

### enabled_tools not enforced server-side
- **Failure mode**: LLM can call any tool regardless of the conversation's `enabled_tools` list — prompt injection could trigger unauthorized tool execution
- **Prevention**: Check `conversation.enabled_tools` before every tool execution in the orchestrator loop
