# Event-Driven Architecture Refactoring — Attendee EMS

> **Date:** April 17, 2026
> **Context:** OVH → GCP migration, agency call preparation
> **Goal:** Make async operations event-driven with Redis + BullMQ, deploy a pgBouncer sidecar (dev-owned), and separate DB pools

---

## Table of Contents

1. [Current Architecture — Overview](#1-current-architecture)
2. [Identified Problems](#2-identified-problems)
3. [Target Architecture](#3-target-architecture)
4. [Phased Refactoring Plan](#4-phased-refactoring-plan)
5. [Effort Estimation](#5-effort-estimation)
6. [Discussion Points with GCP Agency](#6-discussion-points-with-gcp-agency)

---

## 1. Current Architecture

### 1.1 Overview

```
┌─────────────────────────────────────────────────────────┐
│                    VM (Docker Compose)                  │
│                                                         │
│  ┌──────────┐     ┌──────────────────────────────────┐  │
│  │  Nginx   │────▶│         NestJS API               │  │
│  │          │     │                                  │  │
│  │ - Reverse│     │  ┌─── Single Thread (Event Loop) │  │
│  │   proxy  │     │  │                               │  │
│  │ - Static │     │  │  Check-in ──────────────┐     │  │
│  │   front  │     │  │  Import Excel ──────────┤     │  │
│  │ - Rate   │     │  │  Badge Gen ─────────────┤     │  │
│  │   limit  │     │  │  Email ─────────────────┤     │  │
│  └──────────┘     │  │  Scoring ───────────────┤     │  │
│                   │  │  Webhook n8n ───────────┤     │  │
│                   │  │  WebSocket emit ────────┘     │  │
│                   │  │                               │  │
│                   │  │  EVERYTHING on the same thread│  │
│                   │  │  NO queue                     │  │
│                   │  │  NO Redis                     │  │
│                   │  └───────────────────────────────│  │
│                   │                                  │  │
│                   │  Puppeteer/Chromium (process)    │  │
│                   │  └─ PDF export only              │  │
│                   └──────────┬───────────────────────┘  │
│                              │                          │
│  ┌───────────────────────────▼─────────────────────┐    │
│  │    PostgreSQL 16 (currently on the VM)          │    │
│  │    Prisma pool: ~10 connections (default)       │    │
│  │    NO pgBouncer                                 │    │
│  │    NO separate pool                             │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

       Local machines                    Mobile devices
  ┌───────────────────┐         ┌──────────────────┐
  │ EMS Print Client  │◀── WS ──│   Mobile App     │
  │ (Electron)        │         │   (Expo/RN)      │
  │ Polling + print   │         │   QR scan        │
  └───────────────────┘         └──────────────────┘
```

### 1.2 Current Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Backend | NestJS | 10.x |
| ORM | Prisma | 6.17 |
| Database | PostgreSQL | 16 |
| Real-time | Socket.IO | 4.x |
| Email | Nodemailer (SMTP) | — |
| Badge PDF | Puppeteer/Chromium | — |
| Badge printing | HTML direct (via Print Client) | — |
| File storage | Cloudflare R2 | — |
| Queue system | **NONE** | — |
| Cache | **NONE** | — |
| Connection pooler | **NONE** | — |

### 1.3 Current PrismaService

```typescript
// src/infra/db/prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor(private configService: ConfigService) {
    super({
      datasources: {
        db: { url: configService.databaseUrl },
      },
    });
  }
}
```

**Issues:**
- Single instance, single pool (~10 connections by default)
- No separation between critical and bulk operations
- `connection_limit` not configured
- No explicit `pool_timeout`

### 1.4 Current Check-in Flow (most critical operation)

```
POST /registrations/{id}/check-in
│
├─ AWAIT  prisma.registration.findFirst()                    ~5ms DB
├─ AWAIT  prisma.registration.update({ checked_in_at })      ~3ms DB
├─ AWAIT  eventTablesService.autoAssignOnCheckIn()            ~10ms DB (3-4 queries, NO transaction ⚠️)
├─ AWAIT  prisma.registration.findUnique() (if table)        ~3ms DB
├─ SYNC   websocketGateway.emitRegistrationCheckedIn()        ~0.1ms
├─ FIRE   n8nWebhookService.notifyCheckIn().catch(() => {})   5s timeout, no retry, errors swallowed
└─ FIRE   recalculateAttendeeMetrics().catch(() => {})        complex SQL query + transaction, errors swallowed
    │
    ▼
HTTP response sent (~25ms)
The 2 fire-and-forget operations continue in background on the same thread
```

### 1.5 Complete Inventory of Fire-and-Forget Operations

**18 occurrences in `registrations.service.ts`:**

| # | Operation | Trigger | Pattern |
|---|-----------|---------|---------|
| 1 | `recalculateAttendeeMetrics()` | updateStatus | `.catch(() => {})` |
| 2 | `recalculateAttendeeMetrics()` | create | `.catch(() => {})` |
| 3-4 | `recalculateAttendeeMetrics()` ×2 | update (new + old attendee) | `.catch(() => {})` |
| 5 | `recalculateAttendeeMetrics()` | merge | `.catch(() => {})` |
| 6 | `recalculateAttendeeMetrics()` | restore | `.catch(() => {})` |
| 7 | `recalculateAttendeeMetrics()` | permanentDelete | `.catch(() => {})` |
| 8-9 | `recalculateAttendeeMetrics()` ×N | **bulkImport forEach** | `.catch(() => {})` in loop |
| 10 | `n8nWebhookService.notifyCheckIn()` | **check-in** | `.catch(() => {})` |
| 11 | `recalculateAttendeeMetrics()` | **check-in** | `.catch(() => {})` |
| 12 | `recalculateAttendeeMetrics()` | undoCheckIn | `.catch(() => {})` |
| 13 | `recalculateAttendeeMetrics()` | checkOut | `.catch(() => {})` |
| 14 | `recalculateAttendeeMetrics()` | undoCheckOut | `.catch(() => {})` |
| 15-16 | `recalculateAttendeeMetrics()` ×N | **bulkCheckIn forEach** | `.catch(() => {})` in loop |
| 17-18 | `recalculateAttendeeMetrics()` ×N | bulkDelete forEach | `.catch(() => {})` in loop |

**Common issues:**
- All errors are **silently swallowed** (`.catch(() => {})`)
- `forEach` loops launch N recalculations in uncontrolled parallel (import of 3000 rows = 3000 simultaneous recalculations)
- No retry, no tracking, no visibility
- Massive DB pressure during bulk operations

### 1.6 Scoring/Affinities State

`recalculateAttendeeMetrics()` actually performs **2 heavy operations**:

**Scoring** (`attendee-scoring.service.ts`):
- 1 complex SQL query with LATERAL joins (participant stats)
- 1 attendee read
- 1 score computation
- 1 attendee UPDATE
- **~4 queries per call**

**Affinities** (`attendee-affinity.service.ts`):
- 1 massive raw SQL with LATERAL joins (affinities by tag)
- 1 `deleteMany` (deletes all existing affinities)
- N `create` (recreates new affinities)
- All within a **transaction**
- **~3-10 queries per call depending on tag count**

**Impact of a 1000-row import:** 1000 × (4 + 3-10) = **7,000 to 14,000 fire-and-forget queries launched in parallel on the same DB pool** 💥

### 1.7 Table Auto-Assignment State (check-in)

```typescript
// event-tables.service.ts — autoAssignOnCheckIn()
// 3-4 SEPARATE queries, NOT in a transaction ⚠️
1. prisma.registration.findUnique()       // Read table preferences
2. prisma.eventSetting.findUnique()       // Read fill mode
3. prisma.eventTable.findFirst()          // Find available table (×N if multiple)
4. prisma.registration.update()           // Assign the table
```

**Race condition issue:** If 2 check-ins arrive simultaneously for the same table, both can read the same available capacity and assign beyond the limit.

### 1.8 Email State

- Nodemailer via synchronous SMTP (3-5s per email)
- Called in a blocking manner within approval/rejection flows
- No retry, no queue
- If SMTP timeout → the client waits

### 1.9 Print Queue State

- Printer registry **in-memory** (lost on container restart)
- Print jobs stored in DB but no worker processing them
- Relies entirely on the Electron Print Client polling via WebSocket
- Works well on single-VM but **incompatible with horizontal scaling**

---

## 2. Identified Problems

### 2.1 Problem Summary by Criticality

| # | Problem | Criticality | Impact |
|---|---------|-------------|--------|
| 1 | Single DB pool, no pgBouncer | 🔴 CRITICAL | Import blocks check-ins |
| 2 | 18 fire-and-forget `.catch(() => {})` | 🔴 CRITICAL | Uncontrolled DB flooding (bulk), invisible errors |
| 3 | Bulk import launches N parallel recalculations | 🔴 CRITICAL | 1000 rows = 7000-14000 parallel queries |
| 4 | Table auto-assign without transaction | 🟡 IMPORTANT | Race condition on capacity |
| 5 | Blocking emails (sync SMTP) | 🟡 IMPORTANT | 3-5s user latency |
| 6 | n8n webhook without retry | 🟡 IMPORTANT | Data silently lost |
| 7 | Printer registry in-memory | 🟡 IMPORTANT | Lost on restart, incompatible with multi-VM |
| 8 | No cache | 🟢 MINOR | Unnecessary repeated DB queries |
| 9 | No application monitoring | 🟢 MINOR | No visibility into problems |

### 2.2 The Disaster Scenario (current)

```
Tenant A: 2000-person event, check-in starts
Tenant B: Admin imports 3000 Excel rows

1. Tenant B's import begins
   → Parse Excel (blocks the event loop ~1-2s)
   → 3000 sequential inserts (monopolizes DB pool)
   
2. At the end of the import:
   → 3000 × recalculateAttendeeMetrics() launched in parallel
   → = 3000 × 7-14 queries = 21,000 to 42,000 queries
   → ALL on the same DB pool (10 connections)
   → Pool saturated immediately

3. Meanwhile, Tenant A's check-ins:
   → Connection pool timeout
   → Hostesses see errors
   → WebSockets disconnect
   → Printing stops
   → 💥 Event crashed
```

---

## 3. Target Architecture

### 3.1 Overview

```
┌───────────────────────────────────────────────────────────────────┐
│                    GCP VM (Docker Compose)                        │
│                                                                   │
│  ┌──────────┐     ┌──────────────────────────────────────────┐   │
│  │  Nginx   │────▶│         NestJS API                       │   │
│  │          │     │                                          │   │
│  │ - Reverse│     │  Main Thread (Event Loop)                │   │
│  │   proxy  │     │  ┌───────────────────────────────────┐   │   │
│  │ - Static │     │  │ Check-in ────▶ Fast HTTP response │   │   │
│  │   front  │     │  │    └─ emit event → Redis Queue    │   │   │
│  └──────────┘     │  │                                   │   │   │
│                   │  │ API CRUD ──▶ Main pool (15)        │   │   │
│                   │  │ WebSocket ──▶ Redis Adapter        │   │   │
│                   │  └───────────────────────────────────┘   │   │
│                   │                                          │   │
│                   │  BullMQ Workers (same process)            │   │
│                   │  ┌───────────────────────────────────┐   │   │
│                   │  │ scoring.processor ──▶ Bulk pool (5)│   │   │
│                   │  │ webhook.processor ──▶ HTTP (retry) │   │   │
│                   │  │ email.processor ──▶ SMTP (retry)   │   │   │
│                   │  │ import.processor ──▶ Bulk pool (5) │   │   │
│                   │  └───────────────────────────────────┘   │   │
│                   └──────────┬───────────────────────────────┘   │
│                              │                                    │
│  ┌───────────────┐  ┌───────▼──────────────────────────────┐    │
│  │    Redis 7    │  │        Cloud SQL PostgreSQL           │    │
│  │               │  │                                      │    │
│  │ - BullMQ jobs │  │  pgBouncer (Docker sidecar, dev-owned)│    │
│  │ - Socket.IO   │  │  ┌──────────┐  ┌──────────────┐     │    │
│  │   adapter     │  │  │ Main     │  │ Bulk pool    │     │    │
│  │ - Printer     │  │  │ pool     │  │ (imports,    │     │    │
│  │   registry    │  │  │ (15 conn)│  │  scoring)    │     │    │
│  │ - Cache       │  │  │          │  │ (5 conn)     │     │    │
│  │   (optional)  │  │  └──────────┘  └──────────────┘     │    │
│  └───────────────┘  └──────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2 Target Check-in Flow

```
POST /registrations/{id}/check-in
│
├─ AWAIT  prisma.main.registration.findFirst()                   ~5ms
├─ AWAIT  prisma.main.$transaction([                             ~8ms (atomic)
│           registration.update({ checked_in_at }),
│           table auto-assign (if configured)
│         ])
├─ SYNC   websocketGateway.emitRegistrationCheckedIn()           ~0.1ms
├─ QUEUE  scoringQueue.add('recalculate', { attendeeId })        ~1ms (Redis)
├─ QUEUE  webhookQueue.add('n8n-checkin', { registration })      ~1ms (Redis)
│
▼
HTTP response sent (~15ms) ✅ Faster than before

In background (BullMQ worker, separate bulk pool):
  → scoringQueue processor: recalculates score + affinities
  → webhookQueue processor: POST to n8n with 3 exponential retries
```

**Gains:**
- Faster check-in response (~15ms vs ~25ms)
- Scoring and webhook can NEVER block the check-in
- Automatic retry on webhooks
- Full visibility via Bull Board (job monitoring UI)
- Main DB pool is NEVER touched by background jobs

### 3.3 Target Bulk Import Flow

```
POST /registrations/import
│
├─ Parse Excel in a Worker Thread (does not block the event loop)
├─ AWAIT  prisma.bulk.registration.createMany() in batches of 50
│         └─ with adaptivePause() between each batch (CPU throttle)
├─ For each affected attendee:
│    QUEUE  scoringQueue.add('recalculate', { attendeeId })     ~1ms
│    └─ Jobs stack up in Redis
│
▼
HTTP response: { total: 3000, created: 2850, errors: [...] }

In background:
  → The 3000 scoring jobs are processed 1 by 1 by the worker
  → Using the bulk pool (5 connections)
  → At a pace the system can sustain
  → If CPU > 70% → the worker pauses automatically
  → 0 impact on check-ins
```

### 3.4 Target Tech Stack

| Component | Current | Target | Why |
|-----------|---------|--------|-----|
| Queue system | None | **BullMQ** (via `@nestjs/bullmq`) | Reliable queue, retry, backoff, monitoring |
| Cache / Broker | None | **Redis 7** | Backend for BullMQ + Socket.IO adapter + cache |
| Connection pooler | None | **pgBouncer** (Docker container, dev-managed) | DB connection pooling |
| DB pool | 1 pool (~10 conn) | **2 pools** (15 main + 5 bulk) | Isolate check-ins from heavy operations |
| Socket.IO adapter | In-memory | **Redis adapter** | Ready for multi-VM |
| Printer registry | In-memory | **Redis hash** | Persistent, ready for multi-VM |
| Fire-and-forget | `.catch(() => {})` | **BullMQ jobs** | Retry, monitoring, throttle |
| Table auto-assign | 4 separate queries | **Prisma transaction** | Atomicity, no race condition |
| Email | Sync SMTP | **BullMQ email queue** | Non-blocking, retry |
| Excel parsing | Event loop | **Worker Thread** | Does not block the API |

### 3.5 BullMQ Queues to Create

| Queue | Jobs | Concurrency | Retry | DB Pool |
|-------|------|-------------|-------|---------|
| `scoring` | `recalculate-metrics` | 3 | 2 retries, backoff 5s | bulk |
| `webhook` | `n8n-checkin` | 5 | 3 retries, exponential backoff | — |
| `email` | `send-email` | 3 | 3 retries, backoff 30s | — |
| `import` | `process-import-batch` | 1 | 1 retry | bulk |
| `badge-pdf` | `generate-pdf` | 1 | 2 retries | bulk |

---

## 4. Phased Refactoring Plan

### Phase 0 — Redis Infrastructure (GCP agency)

**Who:** GCP Agency
**What:** Provision Redis and expose connection details. **pgBouncer is no longer in scope here** — it will be deployed by the backend developer as a Docker sidecar (see Phase 1).

**Actions:**
- [ ] Provision a 1 GB Memorystore Redis (or accept Redis on the VM via Docker)
- [ ] Expose environment variable: `REDIS_URL`
- [ ] Firewall: allow the VM to access Redis (if Memorystore)
- [ ] Monitoring: add alerts on Redis (memory, connections)
- [ ] Confirm the VM has enough RAM for an additional pgBouncer container (~50 MB)

**Estimated duration:** 0.5-1 day (agency-side infrastructure)

---

### Phase 1 — Separate DB Pool + pgBouncer Sidecar (developer)

**Who:** Backend developer
**Prerequisites:** Phase 0 completed (Redis available, Cloud SQL reachable)
**Risk:** Low (no business logic changes)

> **Note:** Cloud SQL does NOT ship with a built-in pgBouncer. We deploy it ourselves as a Docker container in `docker-compose.prod.yml`, sitting between the NestJS API and Cloud SQL. This keeps connection pooling fully under the dev team's control (config tuning, pool mode, version upgrades) without depending on the agency for every change.

**Target docker-compose.prod.yml:**
```yaml
services:
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  pgbouncer:
    image: edoburu/pgbouncer:1.23.1
    restart: unless-stopped
    environment:
      - DB_HOST=${CLOUDSQL_HOST}        # private IP of Cloud SQL
      - DB_PORT=5432
      - DB_USER=${CLOUDSQL_USER}
      - DB_PASSWORD=${CLOUDSQL_PASSWORD}
      - DB_NAME=${CLOUDSQL_DB}
      - POOL_MODE=transaction            # required for Prisma
      - MAX_CLIENT_CONN=200
      - DEFAULT_POOL_SIZE=25             # 15 main + 5 bulk + headroom
      - SERVER_RESET_QUERY=DISCARD ALL
    healthcheck:
      test: ["CMD", "pg_isready", "-h", "localhost", "-p", "5432"]
      interval: 10s
      timeout: 3s
      retries: 3

  api:
    environment:
      - REDIS_URL=redis://redis:6379
      # API talks to pgbouncer, NOT directly to Cloud SQL
      - DATABASE_URL=postgresql://${CLOUDSQL_USER}:${CLOUDSQL_PASSWORD}@pgbouncer:5432/${CLOUDSQL_DB}?pgbouncer=true&connection_limit=20
    depends_on:
      redis:
        condition: service_healthy
      pgbouncer:
        condition: service_healthy

  nginx:
    # unchanged

volumes:
  redis_data:
```

**Actions:**

#### 1.1 — Refactor PrismaService with 2 pools

```typescript
// src/infra/db/prisma.service.ts
@Injectable()
export class PrismaService implements OnModuleInit, OnModuleDestroy {
  /** Main pool: check-in, real-time API, WebSocket */
  readonly main: PrismaClient;

  /** Secondary pool: imports, scoring, PDF exports */
  readonly bulk: PrismaClient;

  constructor(private configService: ConfigService) {
    const baseUrl = configService.databaseUrl;

    this.main = new PrismaClient({
      datasources: {
        db: { url: `${baseUrl}&connection_limit=15&pool_timeout=15` },
      },
    });

    this.bulk = new PrismaClient({
      datasources: {
        db: { url: `${baseUrl}&connection_limit=5&pool_timeout=30` },
      },
    });
  }

  async onModuleInit() {
    await this.main.$connect();
    await this.bulk.$connect();
  }

  async onModuleDestroy() {
    await this.main.$disconnect();
    await this.bulk.$disconnect();
  }
}
```

#### 1.2 — Migrate existing services

**Global replacement:** `this.prisma.` → `this.prisma.main.` in all services.
Then, mark bulk operations with `this.prisma.bulk.`:

| Service | Method | Pool |
|---------|--------|------|
| RegistrationsService | checkIn, create, update, findAll... | `main` |
| RegistrationsService | bulkImport, bulkDelete, bulkCheckIn | `bulk` |
| AttendeeScoringService | recalculateScore | `bulk` |
| AttendeeAffinityService | recalculateAffinities | `bulk` |
| BadgeGenerationService | generateBadge (PDF) | `bulk` |
| All other services | Standard CRUD | `main` |

#### 1.3 — Atomic transaction for table auto-assign

```typescript
// event-tables.service.ts
async autoAssignOnCheckIn(registrationId, eventId, orgId) {
  return this.prisma.main.$transaction(async (tx) => {
    const registration = await tx.registration.findUnique({ ... });
    if (registration.assigned_table_id) return registration.assigned_table_id;

    const settings = await tx.eventSetting.findUnique({ ... });
    const table = await tx.eventTable.findFirst({ ... }); // With implicit lock

    if (table) {
      await tx.registration.update({
        where: { id: registrationId },
        data: { assigned_table_id: table.id },
      });
      return table.id;
    }
    return null;
  });
}
```

**Estimated duration:** 2-3 days
**Testing:** Verify that simultaneous check-in + import no longer block each other

---

### Phase 2 — BullMQ + Event-Driven Queues (developer)

**Who:** Backend developer
**Prerequisites:** Phase 0 (Redis) + Phase 1 (separate pools)
**Risk:** Moderate (async pattern change)

**Actions:**

#### 2.1 — Install dependencies

```bash
npm install @nestjs/bullmq bullmq
```

#### 2.2 — Configure BullMQ in AppModule

```typescript
// app.module.ts
@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: process.env.REDIS_HOST || 'redis',
        port: parseInt(process.env.REDIS_PORT || '6379'),
      },
      defaultJobOptions: {
        removeOnComplete: 1000,  // Keep last 1000 completed jobs
        removeOnFail: 5000,      // Keep last 5000 failed jobs
      },
    }),
    BullModule.registerQueue(
      { name: 'scoring' },
      { name: 'webhook' },
      { name: 'email' },
    ),
    // ... other modules
  ],
})
```

#### 2.3 — Create the scoring processor

```typescript
// src/jobs/scoring.processor.ts
@Processor('scoring')
export class ScoringProcessor extends WorkerHost {
  constructor(
    private scoringService: AttendeeScoringService,
    private affinityService: AttendeeAffinityService,
    private cpuMonitor: CpuMonitorService,
  ) { super(); }

  async process(job: Job<{ attendeeId: string }>) {
    // CPU throttle: wait if the system is under load
    while (this.cpuMonitor.getUsage() > 70) {
      await new Promise(r => setTimeout(r, 1000));
    }

    const { attendeeId } = job.data;
    await this.scoringService.recalculateScore(attendeeId);
    await this.affinityService.recalculateAffinities(attendeeId);
  }
}
```

#### 2.4 — Create the webhook processor

```typescript
// src/jobs/webhook.processor.ts
@Processor('webhook')
export class WebhookProcessor extends WorkerHost {
  constructor(private n8nService: N8nWebhookService) { super(); }

  async process(job: Job<{ registration: any }>) {
    await this.n8nService.notifyCheckIn(job.data.registration);
    // If it fails → BullMQ automatic retry (3 attempts, exponential backoff)
  }
}
```

#### 2.5 — Create the email processor

```typescript
// src/jobs/email.processor.ts
@Processor('email')
export class EmailProcessor extends WorkerHost {
  constructor(private emailService: EmailService) { super(); }

  async process(job: Job<{ to: string; subject: string; html: string }>) {
    const { to, subject, html } = job.data;
    const success = await this.emailService.sendEmail({ to, subject, html });
    if (!success) throw new Error(`Email failed to ${to}`);
    // BullMQ will auto-retry
  }
}
```

#### 2.6 — Replace fire-and-forget with jobs

**Before:**
```typescript
// registrations.service.ts — check-in
this.n8nWebhookService.notifyCheckIn(result).catch(() => {});
this.recalculateAttendeeMetrics(registration.attendee_id).catch(() => {});
```

**After:**
```typescript
// registrations.service.ts — check-in
await this.scoringQueue.add('recalculate', {
  attendeeId: registration.attendee_id,
});
await this.webhookQueue.add('n8n-checkin', {
  registration: result,
}, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 5000 },
});
```

**Before (bulk import):**
```typescript
for (const attendeeId of affectedAttendeeIds) {
  this.recalculateAttendeeMetrics(attendeeId).catch(() => {});
}
```

**After (bulk import):**
```typescript
const jobs = [...affectedAttendeeIds].map(attendeeId => ({
  name: 'recalculate',
  data: { attendeeId },
}));
await this.scoringQueue.addBulk(jobs);
// The 3000 jobs are queued in Redis in ~50ms
// The worker processes them 1 by 1 at its own pace
```

#### 2.7 — Files to modify

| File | Change | Effort |
|------|--------|--------|
| `registrations.service.ts` | Replace 18 `.catch(() => {})` with `queue.add()` | Medium |
| `n8n-webhook.service.ts` | Remove `.catch()` at call site, keep the service as-is | Low |
| `attendee-scoring.service.ts` | Use `prisma.bulk` instead of `prisma` | Low |
| `attendee-affinity.service.ts` | Use `prisma.bulk` instead of `prisma` | Low |
| `email.service.ts` | Keep as-is, called by the processor | None |
| `app.module.ts` | Add BullMQ config + processor imports | Low |
| **New files** | 3 processors (scoring, webhook, email) | Medium |
| **New file** | CpuMonitorService | Low |

**Estimated duration:** 3-4 days
**Testing:**
- Individual check-in → verify scoring job goes into queue
- 500-row import → verify jobs stack up without flooding DB
- Simulate a failing n8n webhook → verify retry behavior

---

### Phase 3 — Socket.IO Redis Adapter + Printer Registry (developer)

**Who:** Backend developer
**Prerequisites:** Phase 0 (Redis)
**Risk:** Low
**Benefit:** Ready for future multi-VM, persistent printer registry

#### 3.1 — Socket.IO Redis Adapter

```bash
npm install @socket.io/redis-adapter
```

```typescript
// websocket.gateway.ts
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

afterInit(server: Server) {
  const pubClient = createClient({ url: process.env.REDIS_URL });
  const subClient = pubClient.duplicate();
  Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
    server.adapter(createAdapter(pubClient, subClient));
  });
}
```

#### 3.2 — Printer Registry in Redis

```typescript
// print-queue.service.ts
// Replace in-memory Map<> with Redis hashes

async registerPrinter(orgId: string, deviceId: string, printers: PrinterInfo[]) {
  const key = `printers:${orgId}:${deviceId}`;
  await this.redis.set(key, JSON.stringify(printers), 'EX', 300); // TTL 5 min
}

async getExposedPrinters(orgId: string): Promise<PrinterInfo[]> {
  const keys = await this.redis.keys(`printers:${orgId}:*`);
  const results = await Promise.all(keys.map(k => this.redis.get(k)));
  return results.flatMap(r => JSON.parse(r));
}
```

**Estimated duration:** 1-2 days

---

### Phase 4 — Monitoring + Bull Board (developer)

**Who:** Backend developer
**Prerequisites:** Phase 2 (BullMQ)
**Risk:** None

#### 4.1 — Add Bull Board (queue monitoring UI)

```bash
npm install @bull-board/api @bull-board/express @bull-board/nestjs
```

```typescript
// Accessible at /admin/queues (protected by admin auth)
// Visualizes: pending, in-progress, failed, completed jobs
// Allows: retrying failed jobs, flushing a queue, viewing errors
```

#### 4.2 — Add CpuMonitorService (adaptive throttle)

Already documented in the capacity analysis — metrics exposed via `/api/health/metrics`.

**Estimated duration:** 1 day

---

## 5. Effort Estimation

### Summary by Phase

| Phase | Who | Duration | Risk | Prerequisites |
|-------|-----|----------|------|---------------|
| **Phase 0** — Redis (infra) | GCP Agency | 0.5-1 day | Low | GCP migration done |
| **Phase 1** — pgBouncer sidecar + separate DB pool + table transaction | Backend dev | 3-4 days | Low | Phase 0 |
| **Phase 2** — BullMQ event-driven queues | Backend dev | 3-4 days | Moderate | Phase 0 + 1 |
| **Phase 3** — Socket.IO Redis + printer registry | Backend dev | 1-2 days | Low | Phase 0 |
| **Phase 4** — Bull Board monitoring | Backend dev | 1 day | None | Phase 2 |
| **TOTAL developer** | | **9-12 days** | | |

### Recommended Schedule

```
Week 1:
  Mon-Tue: Phase 0 (GCP agency) + Phase 1 started (dev)
  Wed-Thu: Phase 1 completed + testing
  Fri:     Phase 3 (Socket.IO Redis + printer registry)

Week 2:
  Mon-Thu: Phase 2 (BullMQ — the largest phase)
  Fri:     Phase 4 (monitoring) + full integration testing

Week 3:
  Mon-Tue: Load testing, bug fixes
  Wed:     Staging deployment
  Thu-Fri: Validation + production rollout
```

**Total: ~3 weeks including testing and deployment.**

### Phase Dependencies

```
Phase 0 (Redis Infra)
    │
    ├──▶ Phase 1 (pgBouncer Sidecar + Separate DB Pool)
    │        │
    │        └──▶ Phase 2 (BullMQ Queues)
    │                 │
    │                 └──▶ Phase 4 (Monitoring)
    │
    └──▶ Phase 3 (Socket.IO Redis)
         (independent of Phase 1 & 2)
```

### Priority Order if Time is Limited

1. **Phase 0 + Phase 1** (BARE MINIMUM) — Solves the critical DB pool issue
2. **Phase 2** (IMPORTANT) — Eliminates fire-and-forget, adds retry, throttle
3. **Phase 3** (RECOMMENDED) — Prepares for future multi-VM
4. **Phase 4** (NICE TO HAVE) — Operational visibility

---

## 6. Discussion Points with GCP Agency

### What Concerns the Agency (infrastructure)

1. **Redis** — Memorystore Redis (managed) vs Redis on the VM (Docker)?
   - Memorystore: ~$30-50/month, managed, automatic backups
   - Docker Redis on the VM: $0 extra, but shares VM resources
   - **Recommendation:** Docker Redis to start (cost savings), migrate to Memorystore if needed

2. **pgBouncer** — **Owned by the dev team**, deployed as a Docker container alongside the API. The agency only needs to ensure the VM has ~50 MB extra RAM and that Cloud SQL accepts the connection volume.
   - Verify that the Cloud SQL tier supports ~25 inbound connections from pgBouncer
   - Small tier (1 vCPU): ~100 max_connections → OK
   - No agency action needed on the Cloud SQL pooler side

3. **Environment variables** — To expose in CI/CD and on the VM:
   ```
   REDIS_URL=redis://redis:6379            # If Docker on VM
   REDIS_URL=redis://10.x.x.x:6379         # If Memorystore
   CLOUDSQL_HOST=10.x.x.x                  # Private IP of Cloud SQL
   CLOUDSQL_USER=...
   CLOUDSQL_PASSWORD=...
   CLOUDSQL_DB=attendee_ems
   # DATABASE_URL is built dynamically inside docker-compose to point at pgbouncer
   ```

4. **Redis monitoring** — Add alerts on:
   - Redis memory > 80%
   - Redis connections > 100
   - Redis latency > 10ms

5. **Firewall** — If Memorystore: allow the VM to access the Redis VPC

### What Concerns the Developer (code)

- Phases 1 through 4 (everything code-related)
- Load testing after refactoring
- Updating docker-compose.prod.yml

### What Can Be Done in Parallel

```
GCP Agency                          Developer
──────────                          ─────────
OVH → GCP Migration                Phase 1 (DB pool)
Setup Cloud SQL (no pooler          can start locally
needed — dev runs pgBouncer)        with Docker Compose dev
                                    
Provision Redis                     Phase 2 (BullMQ)
Expose REDIS_URL                    develop locally
                                    with Docker Redis
                                    
CI/CD pipeline                      Phase 3 (Socket.IO Redis)
Monitoring/Alerts                   Phase 4 (Bull Board)
                                    
         ──────── Merge ────────
         Integration testing on GCP
         Staging validation
         Production
```

### Questions for the Agency

1. Is Cloud SQL already provisioned? What tier?
2. Does the VM have ~50 MB free RAM for the pgBouncer container? (dev will deploy it)
3. Will the VM have enough RAM for Docker Redis (~256 MB) on top of the API?
4. What's the CI/CD plan? (GitHub Actions → SSH? Cloud Build?)
5. Is there a private VPC network between the VM and Cloud SQL?
6. Is Memorystore Redis being considered or is it out of initial budget?
7. What's the migration timeline? When will the GCP VM be ready?

---

## Appendix: Estimated Additional Cost

| Added Component | Budget Option | Managed Option |
|----------------|---------------|----------------|
| Redis | Docker on the VM ($0) | Memorystore 1 GB (~$35/month) |
| pgBouncer | Docker container on the VM ($0) | — |
| BullMQ | Code only, no infra | — |
| Bull Board | Code only, no infra | — |
| **Total added** | **$0/month** | **~$35/month** |

The majority of the work is **application refactoring**, not additional infrastructure.
