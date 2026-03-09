# RabbitMQ Migration Plan

## Goal

Replace the 1-second polling orchestrator queue with RabbitMQ event-driven message processing. Eliminate multi-server race conditions that cause double messages by leveraging RabbitMQ's single-delivery guarantee.

## Context

- **Current problem**: `orchestrator-worker-tick` polls DB every 1s, multiple servers race to acquire DB locks, causing duplicate processing
- **Solution**: RabbitMQ delivers each message to exactly one consumer — no polling, no locks, no race conditions
- **Branch**: `feat/rabbit-mq`

---

## Phase 1: Infrastructure & RabbitMQ Service

### 1.1 Add RabbitMQ to Docker

- [ ] **`docker-compose.yml`** — Add RabbitMQ service (rabbitmq:4-management-alpine) with ports 5672 + 15672, volume, healthcheck
- [ ] **`docker-compose.cloud.yml`** — Add RabbitMQ service for cloud deployment (same config, no management port exposed). Skip if using a managed service (CloudAMQP, Amazon MQ).

### 1.2 Environment Configuration

- [ ] **`.env.example`** — Add RabbitMQ vars after Redis section:
  - `RABBITMQ_URL` (default: `amqp://hay:hay_password@localhost:5672/`) — single connection string, consistent with how Redis URL works
- [ ] **`server/config/env.ts`** (~line 118) — Add `rabbitmq` config section parsing `RABBITMQ_URL`

### 1.3 RabbitMQ Service (New File)

- [ ] **`server/services/rabbitmq.service.ts`** (NEW) — Core RabbitMQ service:
  - Connection management with reconnection/backoff (similar to redis.service.ts pattern)
  - Channel creation and management
  - Queue/exchange declaration helpers
  - `publish(queue, message)` and `consume(queue, handler)` methods
  - Graceful shutdown (close channels, close connection)
  - Health check method (expose via existing health endpoint)
  - Singleton pattern matching existing services
  - **Connection state events**: emit `connected` / `disconnected` events so other services can react (e.g., enable/disable sweep fallback)

### 1.4 Server Initialization

- [ ] **`server/main.ts`** (~line 29) — Initialize RabbitMQ service after Redis, before scheduler
  - Graceful degradation if RabbitMQ unavailable (like Redis)
  - Declare queues on startup
- [ ] **Health endpoint** — Add RabbitMQ connection status to existing health check response

---

## Phase 2: Orchestrator Queue Migration

### 2.1 Define Queue Topology

Queues to declare on startup:

```
orchestrator.process          — Main processing queue
orchestrator.process.retry    — Delayed retry (dead-letter with TTL)
orchestrator.process.dead     — Failed after max retries (inspect/replay)
```

Message format:

```json
{
  "messageId": "uuid",
  "conversationId": "uuid",
  "organizationId": "uuid",
  "trigger": "customer_message | ai_return | recovery | inactivity",
  "timestamp": "ISO-8601",
  "attempt": 1
}
```

- `messageId`: unique per publish, used for deduplication logging and tracing

### 2.2 Orchestrator Queue Service (New File)

- [ ] **`server/services/orchestrator-queue.service.ts`** (NEW) — Thin wrapper for orchestrator queue operations:
  - `enqueue(conversationId, organizationId, trigger)` — publishes to `orchestrator.process`
  - Handles RabbitMQ unavailability: if disconnected, log warning (the sweep job in 2.5 will catch it)
  - Single place to change queue logic without touching entities or routes

### 2.3 Publish to Queue (Trigger Points)

Every place that currently sets `needs_processing = true` must also call `orchestratorQueue.enqueue()`. Publish from the **service/route layer**, not from entity methods:

- [ ] **Routes/services that call `setProcessed(false)` or `addMessage()` for customer messages** — add `orchestratorQueue.enqueue()` call after the entity method
- [ ] **Routes/services that call `returnToAI()`** — add `orchestratorQueue.enqueue()` call after
- [ ] **`server/orchestrator/index.ts`** (~line 170) — Inactivity warning: enqueue when conversation re-enters processing
- [ ] **`server/services/message-recovery.service.ts`** (lines 97, 112, 147, 168, 187, 289) — All recovery scenarios: enqueue on each recovery attempt

**Important**: Keep setting `needs_processing = true` in DB alongside publishing. This serves as fallback data for the sweep job and debugging.

### 2.4 Consume from Queue (Worker)

- [ ] **`server/workers/orchestrator.worker.ts`** — Refactor:
  - `start()`: Subscribe to `orchestrator.process` queue via RabbitMQ consumer
  - Set prefetch count to **2** (each conversation involves multiple LLM calls taking seconds; tune up based on load testing)
  - On message received: call `orchestrator.processConversation(conversationId)`
  - On success: `channel.ack(msg)`
  - On failure: `channel.nack(msg)` → goes to retry queue with backoff
  - `stop()`: Cancel consumer, wait for in-flight messages to complete
  - Remove `tick()` method

### 2.5 Sweep Safety Net (New Scheduled Job)

- [ ] **`server/services/scheduled-jobs.registry.ts`** — Add `orchestrator-sweep` job (every 30s):
  - Query conversations where `needs_processing = true` AND `last_updated_at < 30s ago` (not recently touched)
  - For each, publish to `orchestrator.process` queue
  - This catches: messages published during RabbitMQ downtime, failed publishes, any edge case where DB flag was set but queue publish was missed
  - Lightweight: only runs a single SELECT, no-op when queue is healthy

### 2.6 Disable Polling

- [ ] **`server/services/scheduled-jobs.registry.ts`** (~line 84) — Set `enabled: false` on `orchestrator-worker-tick` job (don't delete yet — keep as documented fallback)

---

## Phase 3: Job Queue Migration (Deferred)

The job queue service (`job-queue.service.ts`) currently processes jobs inline — there is no actual async dispatching happening. Migrating it to RabbitMQ is premature until jobs are actually dispatched asynchronously.

**Revisit when**: a concrete use case requires async job processing (e.g., bulk document imports, scheduled exports). At that point, add:

```
jobs.process                  — General job processing
jobs.process.dead             — Failed jobs
```

Keep Redis pub/sub for `job:updates` channel (real-time UI broadcast, not work queue).

---

## Phase 4: Cleanup (After Stabilization)

Only proceed after Phase 2 has been running stable in production for at least 1 week.

- [ ] **`server/services/scheduled-jobs.registry.ts`** — Remove `orchestrator-worker-tick` job entirely
- [ ] **`server/repositories/conversation.repository.ts`** (~line 141) — Remove `getAvailableForProcessing()` method
- [ ] **`server/repositories/conversation.repository.ts`** (~line 553) — Remove `acquireLock()` / `releaseLock()` methods
- [ ] **`server/orchestrator/index.ts`** — Remove `loop()` and `getOpenConversations()` methods
- [ ] **`server/orchestrator/run.ts`** — Remove lock acquisition/release from `runConversation()`
- [ ] **Database migration** — Remove columns: `processing_locked_until`, `processing_locked_by`. **Keep `needs_processing`** — it's cheap, powers the sweep safety net, and is invaluable for debugging.
- [ ] **`server/services/stale-message-detector.service.ts`** — Adapt to check RabbitMQ dead-letter queue instead of DB flags

---

## What Stays on Redis (No Change)

| Use Case               | File                                  | Why Redis is correct                     |
| ---------------------- | ------------------------------------- | ---------------------------------------- |
| WebSocket broadcasting | `websocket.service.ts`                | Fan-out to all servers (pub/sub pattern) |
| Conversation events    | `conversation-events.service.ts`      | Fan-out broadcast                        |
| Rate limiting          | `rate-limit.service.ts`               | Atomic counters with TTL                 |
| OAuth state            | `oauth-state.service.ts`              | Ephemeral KV with TTL                    |
| Job progress updates   | `job-queue.service.ts` (pub/sub part) | Real-time UI broadcast                   |

---

## Production Deployment Notes

### Network Architecture

RabbitMQ runs on the **private network only** (never exposed to public internet):

- Same VPC/private network as Postgres and Redis
- Servers connect via internal hostname (e.g., `rabbitmq.internal:5672`)
- Connection string: `amqp://user:password@rabbitmq-host:5672/vhost`
- Enable TLS in production (port 5671)

### Hosting Options

1. **Same server** — Fine for low-to-moderate traffic (~100-200MB RAM overhead)
2. **Dedicated instance** — Small VM/container for isolation and independent scaling
3. **Managed service** — AWS Amazon MQ, CloudAMQP, etc. (least operational overhead)

### High Availability (Future)

- RabbitMQ supports clustering and mirrored queues
- Not needed initially — single node with persistent messages is sufficient
- Consider when scaling beyond 2-3 app servers

### Monitoring

- **RabbitMQ Management UI** (port 15672) — use in dev/staging for queue inspection
- **Health endpoint** — expose queue depth and consumer count via existing `/health` route
- **Dead-letter alerting** — log warning when messages land in `orchestrator.process.dead` (the sweep job can also check DLQ depth)

---

## Dependencies

- **npm package**: `amqplib` + `@types/amqplib`
- **Docker image**: `rabbitmq:4-management-alpine`
- **No new infrastructure** beyond the RabbitMQ container itself

---

## Risk Mitigation

1. **Fallback**: Keep DB polling disabled but present during Phase 2. Can re-enable if RabbitMQ has issues.
2. **Message persistence**: All messages published as `persistent: true` — survives RabbitMQ restarts.
3. **Dead-letter queue**: Failed messages go to DLQ for inspection, not silently lost.
4. **Graceful degradation**: If RabbitMQ is down at startup, server starts with sweep job as the processing fallback. If RabbitMQ disconnects at runtime, sweep job catches orphaned conversations within 30s.
5. **Idempotency**: `processConversation()` is idempotent — the `needs_processing` flag is checked at the start of processing; duplicate queue messages for an already-processed conversation are no-ops.
6. **Sweep safety net**: The 30s sweep job (Phase 2.5) ensures no conversation is permanently stuck, regardless of queue health.

---

## Progress Log

_Update this section as work progresses._

| Date       | Phase    | Status      | Notes                                                                                   |
| ---------- | -------- | ----------- | --------------------------------------------------------------------------------------- |
| 2026-03-09 | Planning | Done        | Plan created, branch `feat/rabbit-mq`                                                   |
| 2026-03-09 | Phase 1  | Done        | Infrastructure, RabbitMQ service, health endpoint, docker-compose                       |
| 2026-03-09 | Phase 2  | Done        | Queue topology, enqueue at all triggers, RabbitMQ consumer, sweep job, polling disabled |
|            | Phase 3  | Deferred    | Job queue not yet async; revisit later                                                  |
|            | Phase 4  | Not started | Cleanup after 1 week stable in production                                               |
