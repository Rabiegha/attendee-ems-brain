# 01 — Code discovery

## Périmètre inspecté

### Dossiers inspectés (backend)

- [src/app.module.ts](../../../../../attendee-ems-back/src/app.module.ts)
- [src/main.ts](../../../../../attendee-ems-back/src/main.ts) — bootstrap, body parser, Sentry
- [src/instrument.ts](../../../../../attendee-ems-back/src/instrument.ts) — init Sentry conditionnelle
- [src/infra/queue/](../../../../../attendee-ems-back/src/infra/queue) — `queue.module.ts`, `queue-names.ts`, `base.processor.ts`, `bull-board.module.ts`, `index.ts`
- [src/infra/storage/](../../../../../attendee-ems-back/src/infra/storage) — `r2.service.ts`, `storage.controller.ts`, `storage.module.ts`
- [src/infra/db/](../../../../../attendee-ems-back/src/infra/db) — `prisma.service.ts`, `prisma.module.ts`
- [src/websocket/](../../../../../attendee-ems-back/src/websocket) — `websocket.gateway.ts`, `websocket.module.ts`
- [src/print-queue/](../../../../../attendee-ems-back/src/print-queue) — `print-queue.controller.ts`, `print-queue.service.ts`, `print-queue.module.ts`
- [src/modules/exports/](../../../../../attendee-ems-back/src/modules/exports) — controller, service, deux processors, where-builders, email attachment
- [src/modules/email/](../../../../../attendee-ems-back/src/modules/email) — `email.service.ts`, templates, controller
- [src/modules/email-templates/](../../../../../attendee-ems-back/src/modules/email-templates)
- [src/modules/registrations/](../../../../../attendee-ems-back/src/modules/registrations) — 4113 lignes
- [src/modules/events/](../../../../../attendee-ems-back/src/modules/events) — 2447 lignes + cron auto-complete + cron purge
- [src/modules/attendees/](../../../../../attendee-ems-back/src/modules/attendees) — 1533 lignes + scoring + affinity
- [src/modules/badge-generation/](../../../../../attendee-ems-back/src/modules/badge-generation) — 2228 lignes (Puppeteer)
- [src/modules/badges/](../../../../../attendee-ems-back/src/modules/badges), [src/modules/badge-templates/](../../../../../attendee-ems-back/src/modules/badge-templates)
- [src/modules/partner-scans/](../../../../../attendee-ems-back/src/modules/partner-scans)
- [src/modules/sessions/](../../../../../attendee-ems-back/src/modules/sessions)
- [src/modules/event-tables/](../../../../../attendee-ems-back/src/modules/event-tables)
- [src/modules/invitations/](../../../../../attendee-ems-back/src/modules/invitations)
- [src/modules/n8n/](../../../../../attendee-ems-back/src/modules/n8n) — service, webhook service, controller (API Key)
- [src/modules/companies/](../../../../../attendee-ems-back/src/modules/companies)
- [src/modules/attendee-types/](../../../../../attendee-ems-back/src/modules/attendee-types)
- [src/modules/tags/](../../../../../attendee-ems-back/src/modules/tags)
- [src/modules/organizations/](../../../../../attendee-ems-back/src/modules/organizations)
- [src/modules/users/](../../../../../attendee-ems-back/src/modules/users)
- [src/modules/roles/](../../../../../attendee-ems-back/src/modules/roles), [src/modules/permissions/](../../../../../attendee-ems-back/src/modules/permissions)
- [src/modules/public/](../../../../../attendee-ems-back/src/modules/public)
- [src/platform/authz/](../../../../../attendee-ems-back/src/platform/authz)
- [src/auth/](../../../../../attendee-ems-back/src/auth)
- [src/common/](../../../../../attendee-ems-back/src/common) — guards, logger, middleware, decorators
- [src/config/](../../../../../attendee-ems-back/src/config) — `config.service.ts`, validation Zod
- [prisma/schema.prisma](../../../../../attendee-ems-back/prisma/schema.prisma) — 1217 lignes
- [prisma/migrations/](../../../../../attendee-ems-back/prisma/migrations) — historique (palier 3a/3b, exports, print-queue…)
- [package.json](../../../../../attendee-ems-back/package.json) — dépendances

### Hors backend (référencé mais non analysé en profondeur)

- `attendee-ems-front/` (RTK Query)
- `attendee-ems-mobile/` (Expo)
- `attendee-ems-print-client/` (Electron + Socket.IO) — voir [07-websocket-print-client.md](07-websocket-print-client.md)
- `Public-Form-Logger/` (service séparé, hors scope)

## Patterns recherchés (et résultats résumés)

| Pattern | Résultat |
|---|---|
| `BullMQ` / `@nestjs/bullmq` / `bullmq` | `FOUND_IN_CODE` — `package.json`, `src/infra/queue/queue.module.ts`, processors exports |
| `Bull Board` | `FOUND_IN_CODE` — `src/infra/queue/bull-board.module.ts`, monté sur `/admin/queues` |
| `ioredis` | `FOUND_IN_CODE` — `package.json` |
| `WebSocketGateway` / Socket.IO | `FOUND_IN_CODE` — `src/websocket/websocket.gateway.ts` (namespace `/events`) |
| `PrintQueue` | `FOUND_IN_CODE` — `src/print-queue/*` + table Prisma `PrintJob` |
| `BadgeGeneration` | `FOUND_IN_CODE` — `src/modules/badge-generation/*` (Puppeteer) |
| `import` (XLSX) | `FOUND_IN_CODE` — `registrations.service.ts bulkImport()` ~700 lignes, lib `xlsx` |
| `export` (CSV/XLSX) | `FOUND_IN_CODE` — module `exports` (BullMQ), endpoints `events/*` (sync ExcelJS), `attendees/bulk-export`, `registrations/bulk-export`, `partner-scans/export*` |
| `bulk` | `FOUND_IN_CODE` — `bulkImport`, `bulkCheckIn`, `bulkUpdateStatus`, `bulkDelete`, `bulkExport`, `bulkRestore`, `bulkPermanentDelete`, `generateBulk`, `generate-badges-bulk` |
| `generate` (badge) | `FOUND_IN_CODE` — `generateBadge`, `generateBulk`, `generateEventBadgesPDF`, `generateBadgePreview` |
| `PDF` | `FOUND_IN_CODE` — Puppeteer dans badge-generation |
| `CSV` | `FOUND_IN_CODE` — `csv-stringify` dans exports processor |
| `XLSX` | `FOUND_IN_CODE` — `xlsx` (import) + `exceljs` (export) |
| `status` | `FOUND_IN_CODE` — enums `EventStatus`, `RegistrationStatus`, `BadgeStatus`, `PrintJobStatus`, `ExportJobStatus`, `InvitationStatus` |
| `progress` | `FOUND_IN_CODE` — uniquement `ExportJob.progress` |
| `retry` | `INFERRED_FROM_CODE` — `attempts: 3` + backoff exponentiel dans `DEFAULT_JOB_OPTIONS` ; endpoint `/print-queue/offline/retry-all` ; pas de champ DB `retryCount` |
| `audit` | `NOT_FOUND` — aucun `AuditLog`/`AuditService` ; substituts partiels : `AttendeeRevision`, `BadgePrint`, soft-deletes |
| `notification` | `INFERRED_FROM_CODE` — pas de modèle ; les notifications passent par WebSocket et email |
| `storage` / `upload` / `download` | `FOUND_IN_CODE` — `R2Service` (Cloudflare R2 via SDK S3), endpoints `storage/upload/test`, `storage/signed-url/:key` |
| `EventEmitter` / `@OnEvent` / `eventEmitter.emit` | `NOT_FOUND` — pas d'EventEmitter2, pas d'event bus interne NestJS |
| `listener` (NestJS) | `NOT_FOUND` |
| `emit` (WebSocket) | `FOUND_IN_CODE` — `emitRegistrationCreated/Updated/CheckedIn`, `emitToOrganization`, `emit('printers:updated')` |
| `queue` (autres que `exports.*`) | `NOT_FOUND` — uniquement deux queues : `exports.attendees`, `exports.registrations` |
| `worker` / `processor` | `FOUND_IN_CODE` — `BaseProcessor`, `ExportAttendeesProcessor`, `ExportRegistrationsProcessor` (in-process, pas de worker séparé) |
| `@Cron` / `@nestjs/schedule` | `FOUND_IN_CODE` — `event-auto-complete.service.ts` (`*/15 * * * *`), `event-purge.service.ts` (`0 5 * * *`) |
| `multer` / `FileInterceptor` | `FOUND_IN_CODE` — `events.controller.ts` (`POST /events/:eventId/registrations/bulk-import`), `storage.controller.ts` (test upload) |
| Sentry | `FOUND_IN_CODE` — `instrument.ts`, `common/logger/sentry-logger.ts`, init conditionnelle si `SENTRY_DSN` |

## Commandes / recherches utilisées

- `list_dir` sur `src/`, `src/modules/`, `src/infra/queue/`, `prisma/`, etc.
- `wc -l` pour identifier les fichiers très volumineux (god-services).
- Inspections ciblées de `app.module.ts`, `package.json`, `queue.module.ts`, `queue-names.ts`, `base.processor.ts`, `bull-board.module.ts`, schémas Prisma, gateway WebSocket, services exports/print-queue/email.
- Trois sous-agents `Explore` (thorough) en parallèle pour : (a) infra async, (b) schéma Prisma + migrations, (c) god-services + coupling.

## Résultats principaux (vue d'avion)

1. **BullMQ déjà présent** mais limité à 2 queues : `exports.attendees`, `exports.registrations`.
2. **Print Queue persisté en DB** (`PrintJob`) avec coordination temps réel via WebSocket. État des imprimantes/Print Client `in-memory` (`Map` côté backend).
3. **Email 100 % synchrone** via Nodemailer (`EmailService.sendEmail()`), pas de queue, pas de batch.
4. **Imports XLSX** entièrement synchrones dans une méthode de **700 lignes** avec une transaction Prisma par ligne (O(N) transactions).
5. **Badge generation** via Puppeteer en mémoire, browser singleton, **séquentiel** par registration en mode bulk.
6. **Exports « lourds » côté `events/*`** (stats, sessions, placements, bulk-export multi-events) sont **synchrones via ExcelJS** dans le contrôleur — `request → buffer → response`.
7. **Aucun EventEmitter2 / event bus interne**, aucun `AuditLog`.
8. **3 god-services** : registrations (4113), events (2447), badges (2228), attendees (1533).
9. **Bull Board** est monté sur `/admin/queues` sans guard NestJS (note dans le code : doit être restreint via Nginx).
10. **Sentry** initialisé conditionnellement, sampling 10 %.
