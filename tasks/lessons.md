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
