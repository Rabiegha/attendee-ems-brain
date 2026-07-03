# 02 — Cartographie des modules backend

Source : [src/app.module.ts](../../../../../attendee-ems-back/src/app.module.ts). Tous les modules ci-dessous sont `FOUND_IN_CODE` (instanciés dans `AppModule`).

## Modules infrastructure

### `QueueModule` — `FOUND_IN_CODE`
- Chemin : [src/infra/queue/queue.module.ts](../../../../../attendee-ems-back/src/infra/queue/queue.module.ts)
- Global. Initialise BullMQ via `BullModule.forRootAsync()` (connexion Redis ioredis, prefix `QUEUE_KEY_PREFIX`, `DEFAULT_JOB_OPTIONS`).
- Pas de controller. Pas de service maison.
- Async potentiel : **fondation actuelle** pour tout job.
- Dépendances : `ConfigService`.

### `BullBoardSetupModule` — `FOUND_IN_CODE`
- Chemin : [src/infra/queue/bull-board.module.ts](../../../../../attendee-ems-back/src/infra/queue/bull-board.module.ts)
- Monte `/admin/queues` (Express adapter). Toggle via `BULL_BOARD_ENABLED`.
- ⚠️ Non protégé par un Guard NestJS (commentaire dans le code).

### `PrismaModule` — `FOUND_IN_CODE`
- [src/infra/db/](../../../../../attendee-ems-back/src/infra/db)
- Service injecté quasi partout. Source de vérité métier.

### `StorageModule` — `FOUND_IN_CODE`
- [src/infra/storage/](../../../../../attendee-ems-back/src/infra/storage)
- `R2Service` (Cloudflare R2 via `@aws-sdk/client-s3` + `@aws-sdk/lib-storage`).
- Méthodes : `uploadFile`, `uploadBadgePdf`, `uploadBadgeImage`, `getFile`, `deleteFile`, `getSignedUploadUrl`, `uploadStream` (multipart), `getSignedDownloadUrl` (72 h), `listFiles`, `cleanupTestBadges`.
- Async potentiel : utilisé par badges + exports processors.

### `WebSocketModule` — `FOUND_IN_CODE`
- [src/websocket/websocket.module.ts](../../../../../attendee-ems-back/src/websocket/websocket.module.ts) — global. Namespace `/events`.
- `EventsGateway` exporté ; injecté par : `RegistrationsService`, `EventsService` (probable), `PrintQueueService`.

### `ConfigModule` — `FOUND_IN_CODE`
- [src/config/](../../../../../attendee-ems-back/src/config) — validation Zod, lecture de toutes les variables (DB, Redis, JWT, R2, SMTP, Sentry, n8n).

### `AuthModule` / `AuthzModule` — `FOUND_IN_CODE`
- [src/auth/](../../../../../attendee-ems-back/src/auth), [src/platform/authz/](../../../../../attendee-ems-back/src/platform/authz)
- JWT access + refresh, RBAC hexagonal (étape 3 selon commentaire), guards `JwtAuthGuard`, `RequirePermissionGuard`.

### `HealthModule` — `FOUND_IN_CODE`
- [src/health/](../../../../../attendee-ems-back/src/health)

## Modules métier

| Module | Chemin | Controller | Service principal | Responsabilité | Liens async actuels / potentiels | Effets secondaires | Statut |
|---|---|---|---|---|---|---|---|
| Organizations | `src/modules/organizations/` | ✔ | `OrganizationsService` | Tenants | — | DB | FOUND_IN_CODE |
| Users | `src/modules/users/` | ✔ | `UsersService` | Comptes | — | DB | FOUND_IN_CODE |
| Roles / Permissions | `src/modules/roles/`, `src/modules/permissions/` | ✔ | — | RBAC | — | DB | FOUND_IN_CODE |
| Invitations | `src/modules/invitations/` | ✔ | `InvitationsService` | Invitation utilisateurs | **Email synchrone** | EmailService | FOUND_IN_CODE |
| Events | `src/modules/events/` | ✔ | `EventsService` (2447 l.) + `EventAutoCompleteService` (cron 15 min) + `EventPurgeService` (cron 05:00) | CRUD events + exports stats/sessions/placements + bulk-export | **Exports sync ExcelJS**, **génération sync** | ExcelJS, Prisma | FOUND_IN_CODE |
| Registrations | `src/modules/registrations/` | ✔ | `RegistrationsService` (4113 l.) | CRUD inscriptions + bulk-import + bulk-checkin + bulk-update-status + scan + check-in + génération badges déléguée | **bulk-import XLSX sync**, **emails sync**, WebSocket, **webhook n8n** | BadgeGen, Email, WS, N8n, Scoring, Affinity | FOUND_IN_CODE |
| Attendees | `src/modules/attendees/` | ✔ | `AttendeesService` (1533 l.) + `AttendeeScoringService` + `AttendeeAffinityService` | CRUD attendees + bulk-export + history | **Export sync** | Scoring, ExcelJS | FOUND_IN_CODE |
| Attendee-types | `src/modules/attendee-types/` | ✔ | `AttendeeTypesService` | CRUD types | — | DB | FOUND_IN_CODE |
| Companies | `src/modules/companies/` | ✔ | `CompaniesService` | CRUD sociétés | — | DB | FOUND_IN_CODE |
| Tags | `src/modules/tags/` | ✔ | `TagsService` | Tags + EventTag (forwardRef avec EventsService) | — | DB | FOUND_IN_CODE |
| Sessions | `src/modules/sessions/` | ✔ | `SessionsService` | Sessions + scan participants (IN/OUT) | Recalc scoring/affinity | Raw SQL `$queryRaw` | FOUND_IN_CODE |
| Event-tables | `src/modules/event-tables/` | ✔ | `EventTablesService` | Tables + placement | — | DB | FOUND_IN_CODE |
| Badge-templates | `src/modules/badge-templates/` | ✔ | `BadgeTemplatesService` | CRUD templates HTML/CSS | — | DB | FOUND_IN_CODE |
| Badge-generation | `src/modules/badge-generation/` | ✔ | `BadgeGenerationService` (2228 l.) | **Génération PDF/image via Puppeteer** + upload R2 | **Puppeteer singleton sync**, **bulk séquentiel** | Puppeteer, R2 | FOUND_IN_CODE |
| Badges | `src/modules/badges/` | ✔ | `BadgesService` | Lecture Badge model | — | DB | FOUND_IN_CODE |
| Email | `src/modules/email/` | ✔ (status + qrcode) | `EmailService` (284 l.) | Nodemailer | **100 % synchrone**, pas de queue | SMTP | FOUND_IN_CODE |
| Email-templates | `src/modules/email-templates/` | ✔ | — | CRUD templates email | — | DB | FOUND_IN_CODE |
| Partner-scans | `src/modules/partner-scans/` | ✔ | `PartnerScansService` | Scan partenaires + exports CSV/XLSX | **Exports sync**, recalc scoring | Scoring, Affinity | FOUND_IN_CODE |
| Public | `src/modules/public/` | ✔ | — | Form public (token public) | Création registrations | DB, peut-être email | FOUND_IN_CODE |
| n8n | `src/modules/n8n/` | ✔ (ApiKey) | `N8nService` (lecture) + `N8nWebhookService` (notif check-in) | Intégration n8n | **Webhook HTTP sync (5 s timeout) catch-and-log** | HttpService | FOUND_IN_CODE |
| Exports | `src/modules/exports/` | ✔ | `ExportsService` + `ExportAttendeesProcessor` + `ExportRegistrationsProcessor` | Exports attendees/registrations | **DÉJÀ BullMQ** (2 queues) | R2 stream, Email (ExportReady) | FOUND_IN_CODE |
| Print-queue | `src/print-queue/` | ✔ | `PrintQueueService` (280 l.) | File d'attente d'impression badges | **WebSocket bidirectionnel**, **état imprimantes en mémoire**, **DB pour jobs** | WS, BadgeGen, Prisma | FOUND_IN_CODE |

## Modules « common »

- `src/common/guards/` : `JwtAuthGuard`, `RequirePermissionGuard`, `PermissionsGuard`, `ApiKeyGuard`, `ThrottlerGuard` (NestJS).
- `src/common/interceptors/`, `src/common/decorators/` (`@RequirePermission`, `@CurrentUser`, `@CurrentOrgId`…), `src/common/middleware/request-logger.middleware.ts`.
- `src/common/logger/sentry-logger.ts` (wrapper Sentry).

## Synthèse couplages remarquables

- **`RegistrationsService` = hub** : injecte 8 services (DB, BadgeGen, WS, Email, EventTables, Scoring, Affinity, N8nWebhook).
- **`PrintQueueService`** : DB + BadgeGen + WS.
- **`ExportsService`** : DB + Email + R2 + 2 queues BullMQ.
- **`BadgeGenerationService`** : DB + R2 + Puppeteer (singleton, init au boot).
- **`InvitationsService`** : DB + Email (sync).
- **God-services** : registrations (4113), events (2447), badge-generation (2228), attendees (1533). Risque d'aggravation si on y empile des `@InjectQueue`.

## Traitements longs potentiels par module

| Module | Action longue identifiée | Sync/async aujourd'hui |
|---|---|---|
| Registrations | `bulkImport` (XLSX → N rows) | **Sync** |
| Registrations | `bulkUpdateStatus` avec emails | **Sync** |
| Registrations | `generateBadgesForEvent` / `generateBadgesBulk` | **Sync, séquentiel** |
| Badge-generation | `generateBulk`, `generateEventBadgesPDF` | **Sync, Puppeteer** |
| Events | `exportEventStats`, `exportEventSessions`, `exportEventPlacements`, `bulkExport` | **Sync, ExcelJS** |
| Attendees | `bulkExport` | **Sync, ExcelJS** |
| Partner-scans | `exportCsv`, `exportExcel` | **Sync** |
| Exports (attendees/registrations) | Export streaming | **Async BullMQ** ✅ |
| Print-queue | Dispatch impressions | **WebSocket**, état mixte mémoire/DB |
| Email | Envoi unitaire et en boucle | **Sync** |
| Invitations | Envoi email d'invitation | **Sync** |
