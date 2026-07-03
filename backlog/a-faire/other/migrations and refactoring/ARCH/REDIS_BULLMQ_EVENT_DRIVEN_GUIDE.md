# Redis + BullMQ Event-Driven Guide (EMS Backend)

Date: 2026-04-24  
Scope: attendee-ems-back

## 1. Why this exists

This guide explains:
- how Redis and BullMQ are implemented in EMS today,
- how the async export pipeline works end-to-end,
- how to add future background jobs in an event-driven style,
- what to monitor in production.

Target audience: any teammate onboarding on background processing.

---

## 2. Current stack and role of each component

- Redis: durable queue backend (BullMQ storage + worker coordination).
- BullMQ: job queue engine (enqueue, retries, backoff, progress).
- NestJS + @nestjs/bullmq: queue integration in modules/providers.
- Bull Board: queue monitoring UI mounted at /admin/queues.
- Prisma ExportJob table: business-level tracking of each export lifecycle.

Important distinction:
- BullMQ Job = technical execution unit.
- ExportJob (DB) = product/business record visible by API/UI.

Both are linked by using the same id (jobId = ExportJob.id).

---

## 3. Source map (files to know)

Core queue infra:
- src/infra/queue/queue.module.ts
- src/infra/queue/queue-names.ts
- src/infra/queue/base.processor.ts
- src/infra/queue/bull-board.module.ts
- src/infra/queue/index.ts

Config:
- src/config/validation.ts
- src/config/config.service.ts

Health:
- src/health/health.controller.ts

Feature using queue (first implementation):
- src/modules/exports/exports.module.ts
- src/modules/exports/exports.service.ts
- src/modules/exports/exports.processor.ts
- src/modules/exports/exports.controller.ts
- src/modules/email/templates/export.template.ts
- prisma/schema.prisma (ExportJob model + enums)

Runtime/deploy:
- docker-compose.dev.yml
- docker-compose.prod.yml
- .env.example

---

## 4. Redis and BullMQ implementation details

### 4.1 Global queue bootstrap

QueueModule is global and configures BullMQ once for all queues:
- Redis connection from ConfigService:
  - REDIS_HOST
  - REDIS_PORT
  - REDIS_PASSWORD
  - REDIS_DB
- prefix from QUEUE_KEY_PREFIX (default: ems)
- default job options from DEFAULT_JOB_OPTIONS

Implementation note:
- maxRetriesPerRequest is set to null for ioredis blocking command compatibility.

### 4.2 Queue naming convention

Queue names are centralized in queue-names.ts.
Current queue:
- exports.attendees

Convention:
- <domain>.<action>
- examples for future: badges.generate, emails.bulk-send, reports.daily-rollup

### 4.3 Default retry behavior

DEFAULT_JOB_OPTIONS:
- attempts: 3
- backoff: exponential, delay 5000ms
- removeOnComplete: keep 1000 jobs
- removeOnFail: keep 5000 jobs

Meaning:
- transient failures retry automatically,
- queue memory is capped,
- failed jobs remain inspectable.

### 4.4 Processor base class

BaseProcessor provides:
- uniform start/success/failure logs,
- typed handle(job) contract,
- protected hooks onSuccess / onFailure.

This avoids duplicating worker boilerplate in each feature processor.

### 4.5 Monitoring UI

Bull Board is mounted at:
- /admin/queues

Toggle:
- BULL_BOARD_ENABLED=true|false

Security warning:
- current setup is not authenticated.
- must be private/protected in production (reverse proxy ACL/basic auth/VPN).

---

## 5. Export pipeline (real flow currently in production code)

### 5.1 API entrypoints

- POST /api/exports/attendees/estimate
- POST /api/exports/attendees
- GET /api/exports/:id

### 5.2 Enqueue flow

When POST /api/exports/attendees is called:
1. ExportsService validates payload.
2. Computes estimate (rows, eta).
3. Creates ExportJob row with status=pending.
4. Enqueues BullMQ job on queue exports.attendees.
5. Returns immediately (HTTP 202 style behavior).

### 5.3 Worker flow

ExportAttendeesProcessor (queue consumer):
1. Loads ExportJob from DB.
2. Sets status=running, progress=0.
3. Streams data batch-by-batch from Prisma.
4. Serializes CSV or XLSX.
5. Uploads stream to R2.
6. Generates signed download URL (TTL 72h).
7. Sets status=completed, progress=100 + file metadata.
8. Sends "export ready" email.

Failure path:
- retries happen automatically.
- on final failed attempt only: status=failed + failed email.

### 5.4 Why this is event-driven enough now

Current model is command-driven async:
- API emits command "create export" via queue add.
- Worker executes side effects off-request.

This already decouples user request latency from heavy processing.

---

## 6. Environment variables used by queue infra

Required for Redis/BullMQ:
- REDIS_HOST
- REDIS_PORT
- REDIS_PASSWORD (optional in dev, required in prod compose)
- REDIS_DB
- QUEUE_KEY_PREFIX
- BULL_BOARD_ENABLED

Related for current export workload:
- R2_ACCOUNT_ID
- R2_ACCESS_KEY_ID
- R2_SECRET_ACCESS_KEY
- R2_BUCKET_NAME
- R2_PUBLIC_URL (used by other storage flows)

Operational gotcha discovered during implementation:
- In dev Docker, API reads .env.docker (not .env).
- Restart is not enough to reload env_file changes in some cases.
- Use force recreate when needed:
  - docker compose -f docker-compose.dev.yml up -d --force-recreate api

---

## 7. Health and observability

### 7.1 Health endpoint

GET /api/health returns redis field:
- ok: ping success
- down: ping failed
- unknown: Redis client not initialized

### 7.2 Logs to watch

Worker lifecycle logs come from BaseProcessor:
- Job <id> started
- Job <id> completed in <ms>
- Job <id> failed (attempt x/y)

### 7.3 Bull Board usage

Use Bull Board to inspect:
- waiting/active/completed/failed jobs,
- payload and attempts,
- retry behavior.

---

## 8. How to add a new async workload (recommended pattern)

Use this exact pattern for any new background process.

### Step 1: declare queue name

Add in queue-names.ts:
- QUEUE_NAMES.<NEW_NAME> = '<domain>.<action>'

### Step 2: register queue in feature module

In module imports:
- BullModule.registerQueue({ name: QUEUE_NAMES.<NEW_NAME> })

### Step 3: producer service

In service:
- inject Queue with @InjectQueue(QUEUE_NAMES.<NEW_NAME>)
- persist a business row first if needed
- enqueue command payload via queue.add()

### Step 4: processor worker

Create processor class:
- @Processor(QUEUE_NAMES.<NEW_NAME>)
- extends BaseProcessor<TData, TResult>
- implement handle(job)
- update business status/progress in DB
- use onFailure for final side effects

### Step 5: expose status endpoint

Provide API to read business job status from DB.
Do not rely on queue internals in frontend.

---

## 9. Event-driven roadmap (next evolution)

Current architecture is queue-command oriented. To scale to many workflows, adopt this layered model:

1) Domain event emission layer
- services emit domain events (ex: attendee.created, registration.checked-in).

2) Outbox layer (recommended)
- write event in DB outbox in same transaction as business mutation.
- dedicated dispatcher publishes outbox events to BullMQ queues.

3) Worker orchestration layer
- each bounded context has its own queue(s) and processors.
- idempotency keys prevent duplicate side effects.

4) Side-effect handlers
- email, PDF, external webhook, analytics, notifications.

Benefits:
- transactional safety,
- replayability,
- easier audit/debug,
- clear separation between sync business write and async effects.

---

## 10. Reliability rules to keep

For every new worker:
- Make handler idempotent (safe on retry).
- Persist state transitions in DB.
- Track progress when job is long.
- Fail loudly in logs with job id and attempt.
- Retry only transient errors.
- Cap job retention (removeOnComplete/removeOnFail).
- Avoid calling external systems before state persistence.

For every producer:
- Validate payload before enqueue.
- Keep payload minimal (ids, not full objects).
- Correlate queue job id with business row id.

---

## 11. Production runbook (quick)

Preflight:
1. Redis up and healthy.
2. REDIS_PASSWORD set in prod.
3. BULL_BOARD disabled or protected.
4. Prisma schema/migrations applied.

Verification:
1. /api/health shows redis=ok.
2. Enqueue a test job.
3. Observe worker logs and Bull Board.
4. Confirm business row reaches completed state.

Incident triage:
1. If jobs stuck waiting: check Redis connectivity and queue name mismatch.
2. If jobs fail immediately: inspect missing env vars and external credentials.
3. If retries exhausted: inspect last error + payload + side-effect provider logs.

---

## 12. Notes specific to current export implementation

- Signed URL TTL is 72 hours.
- Export formats: csv and xlsx.
- Progress is persisted in ExportJob.progress.
- Email expiration display now formats with explicit timezone to avoid UTC drift in human-readable text.

---

## 13. Suggested next improvements

1. Add auth guard in front of /admin/queues.
2. Add dead-letter queue strategy for irrecoverable failures.
3. Add standardized idempotency helper for all processors.
4. Add outbox table + dispatcher as foundation for broader event-driven adoption.
5. Add metrics (queue depth, success rate, retry count, processing latency).
