# Refactoring Architecture Event-Driven — Attendee EMS

> **Date :** 17 avril 2026
> **Contexte :** Migration OVH → GCP, préparation call agence
> **Objectif :** Rendre les opérations async event-driven avec Redis + BullMQ, ajouter pgBouncer, séparer les pools DB

---

## Table des matières

1. [Architecture actuelle — État des lieux](#1-architecture-actuelle)
2. [Problèmes identifiés](#2-problèmes-identifiés)
3. [Architecture cible](#3-architecture-cible)
4. [Plan de refactoring par phases](#4-plan-de-refactoring-par-phases)
5. [Estimation des efforts](#5-estimation-des-efforts)
6. [Points de discussion avec l'agence GCP](#6-points-de-discussion-avec-lagence-gcp)

---

## 1. Architecture actuelle

### 1.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────┐
│                    VM (Docker Compose)                   │
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
│  │          │     │  │  Scoring ───────────────┤     │  │
│  └──────────┘     │  │  Webhook n8n ───────────┤     │  │
│                   │  │  WebSocket emit ────────┘     │  │
│                   │  │                               │  │
│                   │  │  TOUT sur le même thread      │  │
│                   │  │  PAS de queue                 │  │
│                   │  │  PAS de Redis                 │  │
│                   │  └───────────────────────────────│  │
│                   │                                  │  │
│                   │  Puppeteer/Chromium (process)     │  │
│                   │  └─ Export PDF uniquement         │  │
│                   └──────────┬───────────────────────┘  │
│                              │                          │
│  ┌───────────────────────────▼─────────────────────┐    │
│  │    PostgreSQL 16 (sur la VM actuellement)        │    │
│  │    Pool Prisma : ~10 connexions (défaut)         │    │
│  │    PAS de pgBouncer                              │    │
│  │    PAS de pool séparé                            │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

       Machines locales                  Mobiles
  ┌───────────────────┐          ┌──────────────────┐
  │ EMS Print Client  │◀── WS ──│   App Mobile     │
  │ (Electron)        │         │   (Expo/RN)      │
  │ Polling + print   │         │   QR scan        │
  └───────────────────┘         └──────────────────┘
```

### 1.2 Stack technique actuelle

| Composant | Technologie | Version |
|-----------|------------|---------|
| Backend | NestJS | 10.x |
| ORM | Prisma | 6.17 |
| Base de données | PostgreSQL | 16 |
| Temps réel | Socket.IO | 4.x |
| Email | Nodemailer (SMTP) | — |
| Badge PDF | Puppeteer/Chromium | — |
| Badge impression | HTML direct (via Print Client) | — |
| File storage | Cloudflare R2 | — |
| Queue system | **AUCUN** | — |
| Cache | **AUCUN** | — |
| Connection pooler | **AUCUN** | — |

### 1.3 PrismaService actuel

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

**Problèmes :**
- Instance unique, pool unique (~10 connexions par défaut)
- Aucune séparation entre opérations critiques et bulk
- `connection_limit` non configuré
- Pas de `pool_timeout` explicite

### 1.4 Flow check-in actuel (opération la plus critique)

```
POST /registrations/{id}/check-in
│
├─ AWAIT  prisma.registration.findFirst()                    ~5ms DB
├─ AWAIT  prisma.registration.update({ checked_in_at })      ~3ms DB
├─ AWAIT  eventTablesService.autoAssignOnCheckIn()            ~10ms DB (3-4 queries, PAS de transaction ⚠️)
├─ AWAIT  prisma.registration.findUnique() (si table)        ~3ms DB
├─ SYNC   websocketGateway.emitRegistrationCheckedIn()        ~0.1ms
├─ FIRE   n8nWebhookService.notifyCheckIn().catch(() => {})   5s timeout, pas de retry, erreurs avalées
└─ FIRE   recalculateAttendeeMetrics().catch(() => {})        query SQL complexe + transaction, erreurs avalées
    │
    ▼
Réponse HTTP renvoyée (~25ms)
Les 2 fire-and-forget continuent en arrière-plan sur le même thread
```

### 1.5 Inventaire complet des opérations fire-and-forget

**18 occurrences dans `registrations.service.ts` :**

| # | Opération | Déclencheur | Pattern |
|---|-----------|-------------|---------|
| 1 | `recalculateAttendeeMetrics()` | updateStatus | `.catch(() => {})` |
| 2 | `recalculateAttendeeMetrics()` | create | `.catch(() => {})` |
| 3-4 | `recalculateAttendeeMetrics()` ×2 | update (new + old attendee) | `.catch(() => {})` |
| 5 | `recalculateAttendeeMetrics()` | merge | `.catch(() => {})` |
| 6 | `recalculateAttendeeMetrics()` | restore | `.catch(() => {})` |
| 7 | `recalculateAttendeeMetrics()` | permanentDelete | `.catch(() => {})` |
| 8-9 | `recalculateAttendeeMetrics()` ×N | **bulkImport forEach** | `.catch(() => {})` en boucle |
| 10 | `n8nWebhookService.notifyCheckIn()` | **check-in** | `.catch(() => {})` |
| 11 | `recalculateAttendeeMetrics()` | **check-in** | `.catch(() => {})` |
| 12 | `recalculateAttendeeMetrics()` | undoCheckIn | `.catch(() => {})` |
| 13 | `recalculateAttendeeMetrics()` | checkOut | `.catch(() => {})` |
| 14 | `recalculateAttendeeMetrics()` | undoCheckOut | `.catch(() => {})` |
| 15-16 | `recalculateAttendeeMetrics()` ×N | **bulkCheckIn forEach** | `.catch(() => {})` en boucle |
| 17-18 | `recalculateAttendeeMetrics()` ×N | bulkDelete forEach | `.catch(() => {})` en boucle |

**Problèmes communs :**
- Toutes les erreurs sont **silencieusement avalées** (`.catch(() => {})`)
- Les boucles `forEach` lancent N recalculs en parallèle incontrôlé (import de 3000 lignes = 3000 recalculs simultanés)
- Aucun retry, aucun suivi, aucune visibilité
- Pression DB massive en cas de bulk opérations

### 1.6 État du scoring/affinités

Le `recalculateAttendeeMetrics()` fait en réalité **2 opérations lourdes** :

**Scoring** (`attendee-scoring.service.ts`) :
- 1 requête SQL complexe avec LATERAL joins (stats du participant)
- 1 lecture de l'attendee
- 1 calcul de score
- 1 UPDATE de l'attendee
- **~4 queries par appel**

**Affinités** (`attendee-affinity.service.ts`) :
- 1 raw SQL massif avec LATERAL joins (affinités par tag)
- 1 `deleteMany` (supprime toutes les affinités existantes)
- N `create` (recrée les nouvelles affinités)
- Le tout dans une **transaction**
- **~3-10 queries par appel selon le nombre de tags**

**Impact d'un import de 1000 lignes :** 1000 × (4 + 3-10) = **7 000 à 14 000 queries fire-and-forget lancées en parallèle sur le même pool DB** 💥

### 1.7 État de la table auto-assignment (check-in)

```typescript
// event-tables.service.ts — autoAssignOnCheckIn()
// 3-4 queries SÉPARÉES, PAS dans une transaction ⚠️
1. prisma.registration.findUnique()       // Lire les choix de table
2. prisma.eventSetting.findUnique()       // Lire le mode de remplissage
3. prisma.eventTable.findFirst()          // Chercher une table dispo (×N si plusieurs)
4. prisma.registration.update()           // Assigner la table
```

**Problème de race condition :** Si 2 check-ins arrivent en même temps pour la même table, les deux peuvent lire la même capacité disponible et assigner au-delà de la limite.

### 1.8 État de l'email

- Nodemailer en SMTP sync (3-5s par email)
- Appelé de manière bloquante dans les flows d'approbation/rejet
- Pas de retry, pas de file d'attente
- Si SMTP timeout → le client attend

### 1.9 État du print queue

- Registry d'imprimantes **en mémoire** (perdu au redémarrage du container)
- Print jobs stockés en DB mais pas de worker qui les traite
- Repose entièrement sur le Print Client Electron qui poll via WebSocket
- Fonctionne bien en single-VM mais **incompatible avec du scaling horizontal**

---

## 2. Problèmes identifiés

### 2.1 Résumé des problèmes par criticité

| # | Problème | Criticité | Impact |
|---|----------|-----------|--------|
| 1 | Pool DB unique, pas de pgBouncer | 🔴 CRITIQUE | Import bloque les check-ins |
| 2 | 18 fire-and-forget `.catch(() => {})` | 🔴 CRITIQUE | Flood DB incontrôlé (bulk), erreurs invisibles |
| 3 | Import bulk lance N recalculs en parallèle | 🔴 CRITIQUE | 1000 lignes = 7000-14000 queries parallèles |
| 4 | Table auto-assign sans transaction | 🟡 IMPORTANT | Race condition sur la capacité |
| 5 | Emails bloquants (SMTP sync) | 🟡 IMPORTANT | Latence utilisateur de 3-5s |
| 6 | Webhook n8n sans retry | 🟡 IMPORTANT | Données perdues silencieusement |
| 7 | Printer registry en mémoire | 🟡 IMPORTANT | Perdu au redémarrage, incompatible multi-VM |
| 8 | Pas de cache | 🟢 MINEUR | Requêtes DB répétées inutilement |
| 9 | Pas de monitoring applicatif | 🟢 MINEUR | Aucune visibilité sur les problèmes |

### 2.2 Le scénario catastrophe (actuel)

```
Tenant A : Event 2000 personnes, check-in démarre
Tenant B : Admin importe 3000 lignes Excel

1. L'import de Tenant B commence
   → Parse Excel (bloque l'event loop ~1-2s)
   → 3000 inserts séquentiels (monopolise le pool DB)
   
2. À la fin de l'import :
   → 3000 × recalculateAttendeeMetrics() lancés en parallèle
   → = 3000 × 7-14 queries = 21 000 à 42 000 queries
   → TOUTES sur le même pool DB (10 connexions)
   → Pool saturé immédiatement

3. Pendant ce temps, les check-ins de Tenant A :
   → Connection pool timeout
   → Les hôtesses voient des erreurs
   → Les WebSocket se déconnectent
   → Les impressions s'arrêtent
   → 💥 Event planté
```

---

## 3. Architecture cible

### 3.1 Vue d'ensemble

```
┌───────────────────────────────────────────────────────────────────┐
│                    VM GCP (Docker Compose)                        │
│                                                                   │
│  ┌──────────┐     ┌──────────────────────────────────────────┐   │
│  │  Nginx   │────▶│         NestJS API                       │   │
│  │          │     │                                          │   │
│  │ - Reverse│     │  Main Thread (Event Loop)                │   │
│  │   proxy  │     │  ┌───────────────────────────────────┐   │   │
│  │ - Static │     │  │ Check-in ────▶ Réponse HTTP rapide│   │   │
│  │   front  │     │  │    └─ emit event → Redis Queue    │   │   │
│  └──────────┘     │  │                                   │   │   │
│                   │  │ API CRUD ──▶ Pool principal (15)   │   │   │
│                   │  │ WebSocket ──▶ Redis Adapter        │   │   │
│                   │  └───────────────────────────────────┘   │   │
│                   │                                          │   │
│                   │  BullMQ Workers (même process)            │   │
│                   │  ┌───────────────────────────────────┐   │   │
│                   │  │ scoring.processor ──▶ Pool bulk (5)│   │   │
│                   │  │ webhook.processor ──▶ HTTP (retry) │   │   │
│                   │  │ email.processor ──▶ SMTP (retry)   │   │   │
│                   │  │ import.processor ──▶ Pool bulk (5) │   │   │
│                   │  └───────────────────────────────────┘   │   │
│                   └──────────┬───────────────────────────────┘   │
│                              │                                    │
│  ┌───────────────┐  ┌───────▼──────────────────────────────┐    │
│  │    Redis 7    │  │        Cloud SQL PostgreSQL           │    │
│  │               │  │                                      │    │
│  │ - BullMQ jobs │  │  pgBouncer (intégré Cloud SQL)       │    │
│  │ - Socket.IO   │  │  ┌──────────┐  ┌──────────────┐     │    │
│  │   adapter     │  │  │ Pool     │  │ Pool bulk    │     │    │
│  │ - Printer     │  │  │ principal│  │ (imports,    │     │    │
│  │   registry    │  │  │ (15 conn)│  │  scoring)    │     │    │
│  │ - Cache       │  │  │          │  │ (5 conn)     │     │    │
│  │   (optionnel) │  │  └──────────┘  └──────────────┘     │    │
│  └───────────────┘  └──────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2 Flow check-in cible

```
POST /registrations/{id}/check-in
│
├─ AWAIT  prisma.main.registration.findFirst()                   ~5ms
├─ AWAIT  prisma.main.$transaction([                             ~8ms (atomique)
│           registration.update({ checked_in_at }),
│           table auto-assign (si configuré)
│         ])
├─ SYNC   websocketGateway.emitRegistrationCheckedIn()           ~0.1ms
├─ QUEUE  scoringQueue.add('recalculate', { attendeeId })        ~1ms (Redis)
├─ QUEUE  webhookQueue.add('n8n-checkin', { registration })      ~1ms (Redis)
│
▼
Réponse HTTP renvoyée (~15ms) ✅ Plus rapide qu'avant

En arrière-plan (worker BullMQ, pool bulk séparé) :
  → scoringQueue processor : recalcule score + affinités
  → webhookQueue processor : POST n8n avec 3 retries exponentiels
```

**Gains :**
- Réponse check-in plus rapide (~15ms vs ~25ms)
- Scoring et webhook ne peuvent JAMAIS bloquer le check-in
- Retry automatique sur les webhooks
- Visibilité totale via Bull Board (UI de monitoring des jobs)
- Le pool DB principal n'est JAMAIS touché par les jobs

### 3.3 Flow import bulk cible

```
POST /registrations/import
│
├─ Parse Excel dans un Worker Thread (ne bloque pas l'event loop)
├─ AWAIT  prisma.bulk.registration.createMany() par batch de 50
│         └─ avec adaptivePause() entre chaque batch (throttle CPU)
├─ Pour chaque attendee affecté :
│    QUEUE  scoringQueue.add('recalculate', { attendeeId })     ~1ms
│    └─ Les jobs s'empilent dans Redis
│
▼
Réponse HTTP : { total: 3000, created: 2850, errors: [...] }

En arrière-plan :
  → Les 3000 jobs de scoring sont traités 1 par 1 par le worker
  → Avec le pool bulk (5 connexions)
  → Au rythme que le système peut supporter
  → Si CPU > 70% → le worker fait une pause automatique
  → 0 impact sur les check-ins
```

### 3.4 Stack technique cible

| Composant | Actuel | Cible | Pourquoi |
|-----------|--------|-------|----------|
| Queue system | Aucun | **BullMQ** (via `@nestjs/bullmq`) | Queue fiable, retry, backoff, monitoring |
| Cache / Broker | Aucun | **Redis 7** | Backend pour BullMQ + Socket.IO adapter + cache |
| Connection pooler | Aucun | **pgBouncer** (intégré Cloud SQL) | Mutualisation connexions DB |
| Pool DB | 1 pool (~10 conn) | **2 pools** (15 main + 5 bulk) | Isolation check-ins / opérations lourdes |
| Socket.IO adapter | In-memory | **Redis adapter** | Prêt pour multi-VM |
| Printer registry | In-memory | **Redis hash** | Persistant, prêt pour multi-VM |
| Fire-and-forget | `.catch(() => {})` | **BullMQ jobs** | Retry, monitoring, throttle |
| Table auto-assign | 4 queries séparées | **Transaction Prisma** | Atomicité, pas de race condition |
| Email | SMTP sync | **BullMQ email queue** | Non-bloquant, retry |
| Excel parsing | Event loop | **Worker Thread** | Ne bloque pas l'API |

### 3.5 Queues BullMQ à créer

| Queue | Jobs | Concurrency | Retry | Pool DB |
|-------|------|-------------|-------|---------|
| `scoring` | `recalculate-metrics` | 3 | 2 retries, backoff 5s | bulk |
| `webhook` | `n8n-checkin` | 5 | 3 retries, backoff exponentiel | — |
| `email` | `send-email` | 3 | 3 retries, backoff 30s | — |
| `import` | `process-import-batch` | 1 | 1 retry | bulk |
| `badge-pdf` | `generate-pdf` | 1 | 2 retries | bulk |

---

## 4. Plan de refactoring par phases

### Phase 0 — Infra Redis + pgBouncer (agence GCP)

**Qui :** Agence GCP
**Quoi :** Ajouter Redis à l'infra, activer pgBouncer sur Cloud SQL

**Actions :**
- [ ] Provisionner un Memorystore Redis 1 Go (ou Redis sur la VM via Docker)
- [ ] Activer pgBouncer sur l'instance Cloud SQL
- [ ] Exposer les variables d'environnement : `REDIS_URL`, `DATABASE_URL` (avec `?pgbouncer=true`)
- [ ] Firewall : autoriser la VM à accéder au Redis
- [ ] Monitoring : ajouter des alertes sur Redis (mémoire, connexions)

**Durée estimée :** 1-2 jours (infra côté agence)

**docker-compose.prod.yml cible :**
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

  api:
    environment:
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=${DATABASE_URL}?pgbouncer=true&connection_limit=20
    depends_on:
      redis:
        condition: service_healthy
      # postgres supprimé → Cloud SQL externe

  nginx:
    # inchangé

volumes:
  redis_data:
```

---

### Phase 1 — Pool DB séparé + pgBouncer (développeur)

**Qui :** Développeur backend
**Prérequis :** Phase 0 terminée (pgBouncer actif)
**Risque :** Faible (pas de changement de logique métier)

**Actions :**

#### 1.1 — Refactorer PrismaService avec 2 pools

```typescript
// src/infra/db/prisma.service.ts
@Injectable()
export class PrismaService implements OnModuleInit, OnModuleDestroy {
  /** Pool principal : check-in, API temps réel, WebSocket */
  readonly main: PrismaClient;

  /** Pool secondaire : imports, scoring, exports PDF */
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

#### 1.2 — Migrer les services existants

**Remplacement global :** `this.prisma.` → `this.prisma.main.` dans tous les services.
Puis, marquer les opérations bulk avec `this.prisma.bulk.` :

| Service | Méthode | Pool |
|---------|---------|------|
| RegistrationsService | checkIn, create, update, findAll... | `main` |
| RegistrationsService | bulkImport, bulkDelete, bulkCheckIn | `bulk` |
| AttendeeScoringService | recalculateScore | `bulk` |
| AttendeeAffinityService | recalculateAffinities | `bulk` |
| BadgeGenerationService | generateBadge (PDF) | `bulk` |
| Tous les autres services | CRUD standard | `main` |

#### 1.3 — Transaction atomique pour table auto-assign

```typescript
// event-tables.service.ts
async autoAssignOnCheckIn(registrationId, eventId, orgId) {
  return this.prisma.main.$transaction(async (tx) => {
    const registration = await tx.registration.findUnique({ ... });
    if (registration.assigned_table_id) return registration.assigned_table_id;

    const settings = await tx.eventSetting.findUnique({ ... });
    const table = await tx.eventTable.findFirst({ ... }); // Avec lock implicite

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

**Durée estimée :** 2-3 jours
**Tests :** Vérifier que check-in + import simultanés ne se bloquent plus

---

### Phase 2 — BullMQ + Queues event-driven (développeur)

**Qui :** Développeur backend
**Prérequis :** Phase 0 (Redis) + Phase 1 (pools séparés)
**Risque :** Modéré (changement de pattern async)

**Actions :**

#### 2.1 — Installer les dépendances

```bash
npm install @nestjs/bullmq bullmq
```

#### 2.2 — Configurer BullMQ dans AppModule

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
        removeOnComplete: 1000,  // Garde les 1000 derniers jobs terminés
        removeOnFail: 5000,      // Garde les 5000 derniers échecs
      },
    }),
    BullModule.registerQueue(
      { name: 'scoring' },
      { name: 'webhook' },
      { name: 'email' },
    ),
    // ... autres modules
  ],
})
```

#### 2.3 — Créer le scoring processor

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
    // Throttle CPU : attend si le système est chargé
    while (this.cpuMonitor.getUsage() > 70) {
      await new Promise(r => setTimeout(r, 1000));
    }

    const { attendeeId } = job.data;
    await this.scoringService.recalculateScore(attendeeId);
    await this.affinityService.recalculateAffinities(attendeeId);
  }
}
```

#### 2.4 — Créer le webhook processor

```typescript
// src/jobs/webhook.processor.ts
@Processor('webhook')
export class WebhookProcessor extends WorkerHost {
  constructor(private n8nService: N8nWebhookService) { super(); }

  async process(job: Job<{ registration: any }>) {
    await this.n8nService.notifyCheckIn(job.data.registration);
    // Si ça échoue → BullMQ retry automatique (3 tentatives, backoff exponentiel)
  }
}
```

#### 2.5 — Créer l'email processor

```typescript
// src/jobs/email.processor.ts
@Processor('email')
export class EmailProcessor extends WorkerHost {
  constructor(private emailService: EmailService) { super(); }

  async process(job: Job<{ to: string; subject: string; html: string }>) {
    const { to, subject, html } = job.data;
    const success = await this.emailService.sendEmail({ to, subject, html });
    if (!success) throw new Error(`Email failed to ${to}`);
    // BullMQ fera un retry automatique
  }
}
```

#### 2.6 — Remplacer les fire-and-forget par des jobs

**Avant :**
```typescript
// registrations.service.ts — check-in
this.n8nWebhookService.notifyCheckIn(result).catch(() => {});
this.recalculateAttendeeMetrics(registration.attendee_id).catch(() => {});
```

**Après :**
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

**Avant (bulk import) :**
```typescript
for (const attendeeId of affectedAttendeeIds) {
  this.recalculateAttendeeMetrics(attendeeId).catch(() => {});
}
```

**Après (bulk import) :**
```typescript
const jobs = [...affectedAttendeeIds].map(attendeeId => ({
  name: 'recalculate',
  data: { attendeeId },
}));
await this.scoringQueue.addBulk(jobs);
// Les 3000 jobs sont empilés dans Redis en ~50ms
// Le worker les traite 1 par 1 à son rythme
```

#### 2.7 — Fichiers à modifier

| Fichier | Changement | Effort |
|---------|-----------|--------|
| `registrations.service.ts` | Remplacer 18 `.catch(() => {})` par `queue.add()` | Moyen |
| `n8n-webhook.service.ts` | Supprimer le `.catch()` au call site, garder le service tel quel | Faible |
| `attendee-scoring.service.ts` | Utiliser `prisma.bulk` au lieu de `prisma` | Faible |
| `attendee-affinity.service.ts` | Utiliser `prisma.bulk` au lieu de `prisma` | Faible |
| `email.service.ts` | Garder tel quel, appelé par le processor | Aucun |
| `app.module.ts` | Ajouter BullMQ config + import des processors | Faible |
| **Nouveaux fichiers** | 3 processors (scoring, webhook, email) | Moyen |
| **Nouveau fichier** | CpuMonitorService | Faible |

**Durée estimée :** 3-4 jours
**Tests :** 
- Check-in individuel → vérifier que le job scoring passe en queue
- Import 500 lignes → vérifier que les jobs s'empilent sans flood DB
- Simuler un webhook n8n qui fail → vérifier le retry

---

### Phase 3 — Socket.IO Redis Adapter + Printer Registry (développeur)

**Qui :** Développeur backend
**Prérequis :** Phase 0 (Redis)
**Risque :** Faible
**Bénéfice :** Prêt pour multi-VM future, printer registry persistant

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

#### 3.2 — Printer Registry dans Redis

```typescript
// print-queue.service.ts
// Remplacer le Map<> en mémoire par Redis hashes

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

**Durée estimée :** 1-2 jours

---

### Phase 4 — Monitoring + Bull Board (développeur)

**Qui :** Développeur backend
**Prérequis :** Phase 2 (BullMQ)
**Risque :** Nul

#### 4.1 — Ajouter Bull Board (UI de monitoring des queues)

```bash
npm install @bull-board/api @bull-board/express @bull-board/nestjs
```

```typescript
// Accessible via /admin/queues (protégé par auth admin)
// Visualise : jobs en attente, en cours, échoués, terminés
// Permet de : retry les jobs failed, vider une queue, voir les erreurs
```

#### 4.2 — Ajouter le CpuMonitorService (throttle adaptatif)

Déjà documenté dans l'analyse de capacité — métriques exposées via `/api/health/metrics`.

**Durée estimée :** 1 jour

---

## 5. Estimation des efforts

### Résumé par phase

| Phase | Qui | Durée | Risque | Prérequis |
|-------|-----|-------|--------|-----------|
| **Phase 0** — Redis + pgBouncer (infra) | Agence GCP | 1-2 jours | Faible | Migration GCP faite |
| **Phase 1** — Pool DB séparé + transaction table | Dev backend | 2-3 jours | Faible | Phase 0 |
| **Phase 2** — BullMQ queues event-driven | Dev backend | 3-4 jours | Modéré | Phase 0 + 1 |
| **Phase 3** — Socket.IO Redis + printer registry | Dev backend | 1-2 jours | Faible | Phase 0 |
| **Phase 4** — Monitoring Bull Board | Dev backend | 1 jour | Nul | Phase 2 |
| **TOTAL développeur** | | **8-11 jours** | | |

### Planning recommandé

```
Semaine 1 :
  Lun-Mar : Phase 0 (agence GCP) + Phase 1 commencée (dev)
  Mer-Jeu : Phase 1 terminée + tests
  Ven     : Phase 3 (Socket.IO Redis + printer registry)

Semaine 2 :
  Lun-Jeu : Phase 2 (BullMQ — la plus grosse phase)
  Ven     : Phase 4 (monitoring) + tests intégration complète

Semaine 3 :
  Lun-Mar : Tests de charge, corrections
  Mer     : Déploiement staging
  Jeu-Ven : Validation + mise en production
```

**Total : ~3 semaines avec tests et déploiement.**

### Dépendances entre phases

```
Phase 0 (Infra Redis + pgBouncer)
    │
    ├──▶ Phase 1 (Pool DB séparé)
    │        │
    │        └──▶ Phase 2 (BullMQ queues)
    │                 │
    │                 └──▶ Phase 4 (Monitoring)
    │
    └──▶ Phase 3 (Socket.IO Redis)
         (indépendant de Phase 1 & 2)
```

### Ordre de priorité si le temps est limité

1. **Phase 0 + Phase 1** (MINIMUM VITAL) — Résout le problème critique du pool DB
2. **Phase 2** (IMPORTANT) — Élimine les fire-and-forget, ajoute retry, throttle
3. **Phase 3** (RECOMMANDÉ) — Prépare le multi-VM futur
4. **Phase 4** (NICE TO HAVE) — Visibilité opérationnelle

---

## 6. Points de discussion avec l'agence GCP

### Ce qui concerne l'agence (infra)

1. **Redis** — Memorystore Redis (managé) vs Redis sur la VM (Docker) ?
   - Memorystore : ~30-50$/mois, managé, backups auto
   - Redis Docker sur la VM : 0$ supplémentaire, mais partage les ressources de la VM
   - **Recommandation :** Redis Docker pour commencer (économie), migration vers Memorystore si besoin

2. **pgBouncer** — Cloud SQL a un pgBouncer intégré, il suffit de l'activer
   - Vérifier que le `connection_limit` du tier Cloud SQL supporte 20 connexions Prisma
   - Petit tier (1 vCPU) : ~100 max_connections → OK
   - Demander : le pooling mode `transaction` est recommandé pour Prisma

3. **Variables d'environnement** — À exposer dans le CI/CD et la VM :
   ```
   REDIS_URL=redis://redis:6379     # Si Docker sur VM
   REDIS_URL=redis://10.x.x.x:6379  # Si Memorystore
   DATABASE_URL=postgresql://...?pgbouncer=true&connection_limit=20
   ```

4. **Monitoring Redis** — Ajouter des alertes sur :
   - Mémoire Redis > 80%
   - Connexions Redis > 100
   - Latence Redis > 10ms

5. **Firewall** — Si Memorystore : autoriser la VM à accéder au VPC Redis

### Ce qui concerne le développeur (code)

- Phase 1 à 4 (tout ce qui est code)
- Tests de charge après refactoring
- Mise à jour du docker-compose.prod.yml

### Ce qui peut être fait en parallèle

```
Agence GCP                          Développeur
──────────                          ────────────
Migration OVH → GCP                Phase 1 (pool DB)
Setup Cloud SQL + pgBouncer         peut commencer en local
Setup réseau, firewall              avec Docker Compose dev
                                    
Provisionner Redis                  Phase 2 (BullMQ)
Exposer REDIS_URL                   développement en local
                                    avec Redis Docker
                                    
CI/CD pipeline                      Phase 3 (Socket.IO Redis)
Monitoring/Alertes                  Phase 4 (Bull Board)
                                    
         ──────── Merge ────────
         Tests intégration sur GCP
         Validation staging
         Production
```

### Questions à poser à l'agence

1. Est-ce que Cloud SQL est déjà provisionné ? Quel tier ?
2. pgBouncer est-il déjà activé ?
3. La VM aura-t-elle assez de RAM pour Redis Docker (~256 Mo) en plus de l'API ?
4. Quel est le plan pour le CI/CD ? (GitHub Actions → SSH ? Cloud Build ?)
5. Y a-t-il un réseau VPC privé entre la VM et Cloud SQL ?
6. Est-ce que Memorystore Redis est envisagé ou hors budget initial ?
7. Quel est le timeline de la migration ? Quand la VM GCP sera prête ?

---

## Annexe : Coût additionnel estimé

| Composant ajouté | Option économique | Option managée |
|-----------------|-------------------|----------------|
| Redis | Docker sur la VM (0$) | Memorystore 1 Go (~35$/mois) |
| pgBouncer | Intégré Cloud SQL (0$) | — |
| BullMQ | Code, pas d'infra | — |
| Bull Board | Code, pas d'infra | — |
| **Total ajouté** | **0$/mois** | **~35$/mois** |

La majorité du travail est du **refactoring applicatif**, pas de l'infrastructure supplémentaire.
