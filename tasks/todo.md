# PR 1 — LLM Provider Adapter (org-level provider + model tiers)

Approved plan: `~/.claude/plans/tidy-twirling-hoare.md`. Research artifact: workflow `wj6c0lukt`.
Deferred metering design: `tasks/managed-llm-metering-design.md` (PR #43).

Branch: `claude/llm-provider-adapter`.

## Slices (build/verify order, one PR)

- [x] **1. Contract + OpenAICompatibleProvider + factory → LLMService (env-default, provably unchanged)** ✅ typecheck/lint/char-test green
- [x] 2. Tiers replace hardcoded models at call sites ✅ (org threading + orchestrator tier labels → folded into slice 5)
- [x] 3. UsageRecord + invokeWithMeta + onUsage seam ✅ (bogus cost log removed in slice 1)
- [x] 4. Capability-driven structured-output rungs + validate-and-repair (ajv) ✅
- [x] 5. OrgLlmConfig in OrganizationSettings jsonb + BYO decryption + bundle cache ✅
- [x] 6. VectorStoreService → EmbeddingProvider + 1536 assertion ✅ (both raw OpenAI clients now gone)
- [x] 7. AnthropicChatProvider (dedicated, @anthropic-ai/sdk) ✅ (streaming stub → slice 9)
- [x] 8. GeminiChatProvider (@google/genai) + Mistral/Grok vendor capability profiles ✅ (gemini chat-only; embeddings managed)
- [x] 9. Streaming (chatStream for all 3 adapters) + AbortSignal-cancel timeout ✅
- [ ] 10. Dashboard org-settings LLM UI

## Acceptance

- No service imports `openai` except the adapter layer.
- Org with no config → behavior byte-for-byte unchanged (gpt-4o default kept).
- Switching provider/model is config-only; call sites never reference model strings.
- Every chat/embedding result carries a normalized UsageRecord; onUsage fires on every call.

## Working notes

- gpt-4o stays the `hard` default (still works for the user); all model IDs env/DB-overridable.
- LLMService is a stateless singleton (12 `new LLMService()` sites, no orgId at construction) →
  resolve per-org inside invoke() by ChatOptions.organizationId.
- Two raw clients: `llm.service.ts:31`, `vector-store.service.ts:47` (already passes dimensions:1536).
- Mirror: git-provider.interface.ts + factory; config-resolver.ts (DB→env→default); encryption.ts decryptValue.
- OrganizationSettings has `[key: string]: unknown` at types/organization-settings.types.ts:311 → no migration.
- Per-model flags (reasoningModel / usesMaxCompletionTokens / acceptsSampling) ride on ChatRequest, set by factory.
