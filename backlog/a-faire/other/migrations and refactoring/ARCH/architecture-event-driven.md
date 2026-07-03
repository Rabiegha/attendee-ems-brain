# Architecture Analysis — Migration Event-Driven

> **Date d'analyse** : 11 mars 2026  
> **Périmètre** : `attendee-ems-back/src/`  
> **Objectif** : Cartographier les dépendances inter-services, les effets de bord, et préparer la migration vers une architecture event-driven (NestJS `@nestjs/event-emitter`).

---

## 1. Inventaire des modules et fichiers de service

### 1.1 Modules enregistrés dans `app.module.ts` (30 imports)

| # | Module | Chemin |
|---|--------|--------|
| 1 | ConfigModule | `src/config/` |
| 2 | PrismaModule | `src/infra/db/` |
| 3 | AuthModule | `src/auth/` |
| 4 | AuthzModule | `src/platform/authz/` |
| 5 | UsersModule | `src/modules/users/` |
| 6 | OrganizationsModule | `src/modules/organizations/` |
| 7 | RolesModule | `src/modules/roles/` |
| 8 | PermissionsModule | `src/modules/permissions/` |
| 9 | InvitationsModule | `src/modules/invitations/` |
| 10 | AttendeesModule | `src/modules/attendees/` |
| 11 | HealthModule | `src/health/` |
| 12 | StorageModule | `src/infra/storage/` |
| 13 | BadgeTemplatesModule | `src/modules/badge-templates/` |
| 14 | BadgeGenerationModule | `src/modules/badge-generation/` |
| 15 | BadgesModule | `src/modules/badges/` |
| 16 | AttendeeTypesModule | `src/modules/attendee-types/` |
| 17 | SessionsModule | `src/modules/sessions/` |
| 18 | EventTablesModule | `src/modules/event-tables/` |
| 19 | EventsModule | `src/modules/events/` |
| 20 | PublicModule | `src/modules/public/` |
| 21 | RegistrationsModule | `src/modules/registrations/` |
| 22 | PartnerScansModule | `src/modules/partner-scans/` |
| 23 | TagsModule | `src/modules/tags/` |
| 24 | WebSocketModule | `src/websocket/` (GLOBAL) |
| 25 | EmailModule | `src/modules/email/` |
| 26 | PrintQueueModule | `src/print-queue/` |

### 1.2 Fichiers service par module

| Module | Service | LOC | Controller |
|--------|---------|-----|------------|
| attendees | `attendees.service.ts` | 885 | `attendees.controller.ts` |
| events | `events.service.ts` | 1 476 | `events.controller.ts` |
| registrations | `registrations.service.ts` | **3 138** | `registrations.controller.ts` |
| badge-generation | `badge-generation.service.ts` | **2 089** | `badge-generation.controller.ts` |
| badge-templates | `badge-templates.service.ts` | 593 | `badge-templates.controller.ts` |
| badges | `badges.service.ts` | 323 | `badges.controller.ts` |
| email | `email.service.ts` | 311 | `email.controller.ts` + `qrcode.controller.ts` |
| invitations | `invitations.service.ts` | 467 | `invitations.controller.ts` |
| organizations | `organizations.service.ts` | 298 | `organizations.controller.ts` |
| users | `users.service.ts` | 325 | `users.controller.ts` |
| partner-scans | `partner-scans.service.ts` | 586 | `partner-scans.controller.ts` |
| event-tables | `event-tables.service.ts` | 482 | `event-tables.controller.ts` |
| sessions | `sessions.service.ts` | 236 | `sessions.controller.ts` |
| tags | `tags.service.ts` | 191 | `tags.controller.ts` |
| attendee-types | `attendee-types.service.ts` | 228 | `attendee-types.controller.ts` |
| permissions | `permissions.service.ts` | 37 | `permissions.controller.ts` |
| roles | `roles.service.ts` | 326 | `roles.controller.ts` |
| public | `public.service.ts` | 417 | `public.controller.ts` |
| print-queue | `print-queue.service.ts` | 280 | `print-queue.controller.ts` |
| auth | `auth.service.ts` | 974 | (dans auth/) |
| **TOTAL** | **20 services** | **~14 320** | |

Modules utilitaires (sans side-effects importants) :
- `config/config.service.ts` (78 LOC)
- `infra/db/prisma.service.ts` (32 LOC)
- `infra/storage/r2.service.ts` (264 LOC)
- `authorization/rbac.service.ts` (141 LOC)
- `platform/authz/core/authorization.service.ts` (143 LOC)

---

## 2. Dépendances injectées par service clé

### 2.1 `RegistrationsService` — LE service le plus couplé (3 138 LOC)

```
Injections :
  ├── PrismaService
  ├── BadgeGenerationService     ← couplage fort
  ├── EventsGateway (WebSocket)  ← couplage fort
  ├── EmailService               ← couplage fort
  └── EventTablesService         ← couplage fort
```

**Side-effects déclenchés :**

| Méthode | Side-effect | Type |
|---------|-------------|------|
| `create()` | `websocketGateway.emitRegistrationCreated()` | WebSocket push |
| `create()` | `emailService.sendRegistrationConfirmationEmail()` | Email (try/catch, non-bloquant) |
| `updateStatus()` | `websocketGateway.emitRegistrationUpdated()` | WebSocket push |
| `approveWithEmail()` | `emailService.sendRegistrationApprovedEmail()` | Email (try/catch) |
| `approveWithEmail()` | Génération QR code URL pour email | Data enrichment |
| `rejectWithEmail()` | `emailService.sendRegistrationRejectedEmail()` | Email (try/catch) |
| `update()` | `websocketGateway.emitRegistrationUpdated()` | WebSocket push |
| `checkIn()` | `eventTablesService.autoAssignOnCheckIn()` | Table auto-assignment |
| `checkIn()` | `websocketGateway.emitRegistrationCheckedIn()` | WebSocket push |
| `undoCheckIn()` | `websocketGateway.emitRegistrationCheckedIn()` | WebSocket push |
| `checkOut()` | `websocketGateway.emitRegistrationUpdated()` | WebSocket push |
| `undoCheckOut()` | `websocketGateway.emitRegistrationUpdated()` | WebSocket push |
| `generateBadge()` | `badgeGenerationService.generateBadge()` | Badge generation (Puppeteer) |
| `generateBadgesForEvent()` | `badgeGenerationService.generateBadge()` × N | Batch badge generation |
| `downloadBadge()` | `badgeGenerationService.generateBadge()` | Badge re-generation |
| `markBadgePrinted()` | Badge print_count increment | DB update |
| `bulkUpdateStatus()` | `emailService.sendRegistrationApprovedEmail()` × N | Batch emails |
| `bulkUpdateStatus()` | `emailService.sendRegistrationRejectedEmail()` × N | Batch emails |

### 2.2 `EventsService` (1 476 LOC)

```
Injections :
  ├── PrismaService
  └── TagsService (forwardRef)   ← couplage faible
```

**Side-effects :** Aucun side-effect direct (pas de WS, pas d'email). Les événements sont purement CRUD. `TagsService` est appelé après la transaction de création pour `updateEventTags()`.

### 2.3 `BadgeGenerationService` (2 089 LOC)

```
Injections :
  ├── PrismaService
  ├── R2Service (S3/Cloudflare)  ← upload fichiers
  └── BadgeTemplatesService      ← résolution template
```

**Side-effects :**
- Lancement de **Puppeteer browser** (singleton) pour rendu HTML→PDF/PNG
- Upload vers **R2 (S3)** des badges générés (PDF + image)
- QR Code generation (base64)
- Gestion de cycle de vie browser (`onModuleDestroy`)

### 2.4 `PrintQueueService` (280 LOC)

```
Injections :
  ├── PrismaService
  ├── BadgeGenerationService     ← re-génère badge avant print
  └── EventsGateway (WebSocket)  ← couplage fort
```

**Side-effects :**

| Méthode | Side-effect |
|---------|-------------|
| `constructor()` | Enregistre callback `onEmsClientDisconnect` sur le gateway |
| `registerPrinters()` | `eventsGateway.emitPrintersUpdated()` → WS push |
| `unregisterDevice()` | `eventsGateway.emitPrintersUpdated()` → WS push |
| `addPrintJob()` | `badgeGenerationService.generateBadge()` → régénération badge |
| `addPrintJob()` | `eventsGateway.emitPrintJobCreated()` → WS push ciblé (device ou org) |
| `updateJobStatus()` | `eventsGateway.emitPrintJobUpdated()` → WS push |

State in-memory : `exposedPrinters: Map<orgId, Map<deviceId, Printer[]>>` — printers exposées par les clients EMS Print connectés via WebSocket.

### 2.5 `EmailService` (311 LOC)

```
Injections : (aucune — standalone)
  └── nodemailer.Transporter (construit dans constructor)
```

**Side-effects :** Envoi SMTP via nodemailer. Service leaf — jamais d'appel vers d'autres services. Template statuses en mémoire (`Map`).

Types d'emails :
- `sendInvitationEmail()`
- `sendPasswordResetEmail()`
- `sendRegistrationConfirmationEmail()`
- `sendRegistrationApprovedEmail()`
- `sendRegistrationRejectedEmail()`

### 2.6 `InvitationsService` (467 LOC)

```
Injections :
  ├── PrismaService
  └── EmailService               ← couplage modéré
```

**Side-effects :**
- `createInvitation()` → `emailService.sendInvitationEmail()`
- `resendInvitation()` → `emailService.sendInvitationEmail()`
- `completeInvitation()` → Création user + TenantUserRole + OrgUser (dans transaction)

### 2.7 `PublicService` (417 LOC)

```
Injections :
  ├── PrismaService
  └── EmailService               ← couplage modéré
```

**Side-effects :**
- `registerToEvent()` → `emailService.sendRegistrationConfirmationEmail()` via `setImmediate()` (asynchrone, hors transaction)

> ⚠️ **Note** : `PublicService` ne passe PAS par `RegistrationsService.create()`. Il duplique la logique de création de registration (upsert attendee + check capacity + check duplicate + create registration + email). C'est un **code smell** important pour la migration.

### 2.8 `OrganizationsService` (298 LOC)

```
Injections :
  └── PrismaService
```

**Side-effects :**
- `create()` → `initializeOrganizationRolesAndPermissions()` (seed des 5 rôles par défaut + permissions RBAC)

### 2.9 `AttendeesService` (885 LOC)

```
Injections :
  └── PrismaService
```

**Side-effects :** Aucun. 100% CRUD + revisions. Pas de WS, pas d'email.

### 2.10 `PartnerScansService` (586 LOC)

```
Injections :
  └── PrismaService
```

**Side-effects :** Aucun. 100% CRUD. Pas de WS, pas d'email.

### 2.11 `UsersService` (325 LOC), `RolesService` (326 LOC), `PermissionsService` (37 LOC)

Tous n'injectent que `PrismaService`. Aucun side-effect.

### 2.12 `EventTablesService` (482 LOC)

```
Injections :
  └── PrismaService
```

Aucun side-effect direct. Fournit `autoAssignOnCheckIn()` consommé par `RegistrationsService`.

### 2.13 `SessionsService` (236 LOC), `TagsService` (191 LOC), `AttendeeTypesService` (228 LOC), `BadgeTemplatesService` (593 LOC)

Tous n'injectent que `PrismaService`. Aucun side-effect notable.

---

## 3. Carte des dépendances inter-services (couplage direct)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  RegistrationsService ──────┬──→ BadgeGenerationService                │
│  (3138 LOC) ★ CENTRAL      │       ├──→ R2Service (S3 upload)          │
│                             │       └──→ BadgeTemplatesService          │
│                             ├──→ EmailService                          │
│                             ├──→ EventsGateway (WebSocket)             │
│                             └──→ EventTablesService                    │
│                                                                         │
│  PrintQueueService ─────────┬──→ BadgeGenerationService                │
│                             └──→ EventsGateway (WebSocket)             │
│                                                                         │
│  InvitationsService ────────┬──→ EmailService                          │
│                             └──→ PrismaService                         │
│                                                                         │
│  PublicService ─────────────┬──→ EmailService                          │
│  (duplique logique          └──→ PrismaService                         │
│   de registration!)                                                     │
│                                                                         │
│  EventsService ─────────────┬──→ TagsService (forwardRef)             │
│                             └──→ PrismaService                         │
│                                                                         │
│  BadgeGenerationService ────┬──→ BadgeTemplatesService                 │
│                             ├──→ R2Service                             │
│                             └──→ PrismaService                         │
│                                                                         │
│  OrganizationsService ──────┬──→ PrismaService                         │
│                             └──→ (seed interne roles/permissions)      │
│                                                                         │
│  [Leaf Services — PrismaService uniquement]                            │
│  AttendeesService, UsersService, RolesService, PermissionsService,     │
│  EventTablesService, SessionsService, TagsService,                     │
│  AttendeeTypesService, BadgeTemplatesService, PartnerScansService,     │
│  BadgesService                                                          │
│                                                                         │
│  [Standalone]                                                           │
│  EmailService (nodemailer, aucune injection de service)                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Résumé couplage par nombre d'injections de services :

| Service | Nb services injectés | Services |
|---------|---------------------|----------|
| RegistrationsService | **4** | BadgeGenerationService, EmailService, EventsGateway, EventTablesService |
| PrintQueueService | **2** | BadgeGenerationService, EventsGateway |
| BadgeGenerationService | **2** | R2Service, BadgeTemplatesService |
| InvitationsService | **1** | EmailService |
| PublicService | **1** | EmailService |
| EventsService | **1** | TagsService |
| Tous les autres | **0** | (PrismaService uniquement) |

---

## 4. WebSocket Gateway — Analyse complète

### Fichier : `src/websocket/websocket.gateway.ts` (246 LOC)

**Module** : `WebSocketModule` (déclaré `@Global()` — accessible partout sans import)

**Namespace** : `/events`

**Authentification** : JWT vérification dans `handleConnection()` → `client.data.user`

**Rooms** : `org:{orgId}` — chaque client rejoint la room de son organisation.

### Événements émis (Server → Client)

| Méthode Gateway | Event name | Consommé par |
|----------------|------------|--------------|
| `emitRegistrationCreated()` | `registration:created` | Front (real-time list) |
| `emitRegistrationUpdated()` | `registration:updated` | Front (real-time list) |
| `emitRegistrationDeleted()` | `registration:deleted` | Front |
| `emitRegistrationCheckedIn()` | `registration:checked-in` | Front + Mobile |
| `emitEventCreated()` | `event:created` | Front |
| `emitEventUpdated()` | `event:updated` | Front |
| `emitEventDeleted()` | `event:deleted` | Front |
| `emitSessionCreated()` | `session:created` | Front |
| `emitSessionUpdated()` | `session:updated` | Front |
| `emitSessionDeleted()` | `session:deleted` | Front |
| `emitAttendeeCreated()` | `attendee:created` | Front |
| `emitAttendeeUpdated()` | `attendee:updated` | Front |
| `emitAttendeeDeleted()` | `attendee:deleted` | Front |
| `emitPrintJobCreated()` | `print-job:created` | EMS Print Client (ciblé par device ou broadcast org) |
| `emitPrintJobUpdated()` | `print-job:updated` | Front + Mobile |
| `emitPrintersUpdated()` | `printers:updated` | Front + Mobile |

### Événements reçus (Client → Server)

| Event name | Handler | Description |
|-----------|---------|-------------|
| `ems-client:identify` | `handleEmsClientIdentify()` | Un EMS Print Client s'identifie avec son `deviceId` |

### EMS Print Client tracking

Le gateway maintient un `Map<socketId, { socketId, orgId, deviceId, connectedAt }>` pour tracker les clients d'impression connectés. Utilisé par `PrintQueueService` pour :
- Savoir si un client est connecté (`isEmsClientConnected()`)
- Cibler l'envoi d'un print job au bon device (`emitPrintJobCreated()` avec routage `printerName::deviceId`)
- Callback de déconnexion pour supprimer les imprimantes du store in-memory

---

## 5. Patterns d'événements existants

### ❌ `@nestjs/event-emitter` — **NON installé, NON utilisé**

Aucune trace de `EventEmitter2`, `@OnEvent`, `EventEmitterModule` dans le code source. Les seules mentions sont dans des documents de planification (`docs/PLAN_IMPLEMENTATION_RBAC_NESTJS.md`, et le présent fichier).

### ✅ Pattern actuel : appels directs synchrones

Le pattern actuel est un **couplage direct par injection de dépendance** :
```typescript
// Dans RegistrationsService.create()
this.websocketGateway.emitRegistrationCreated(orgId, registration);
await this.emailService.sendRegistrationConfirmationEmail({ ... });
```

### ✅ WebSocket comme "event bus" partiel

Le `EventsGateway` fait office de bus de notification vers les clients (front/mobile/print), mais :
- Pas de découplage côté backend — les services appellent le gateway directement
- Pas de pattern pub/sub interne
- Pas de retry/dead-letter

---

## 6. Inventaire complet des side-effects à migrer

### 6.1 Registration Lifecycle (priorité maximale)

| Trigger | Side-effect | Service actuel |
|---------|-------------|----------------|
| Registration created | Email confirmation | RegistrationsService → EmailService |
| Registration created | WebSocket push `registration:created` | RegistrationsService → EventsGateway |
| Registration approved | Email approbation + QR code | RegistrationsService → EmailService |
| Registration rejected | Email refus | RegistrationsService → EmailService |
| Registration status changed | WebSocket push `registration:updated` | RegistrationsService → EventsGateway |
| Registration checked-in | Auto table assignment | RegistrationsService → EventTablesService |
| Registration checked-in | WebSocket push `registration:checked-in` | RegistrationsService → EventsGateway |
| Registration check-out | WebSocket push `registration:updated` | RegistrationsService → EventsGateway |
| Registration updated | WebSocket push `registration:updated` | RegistrationsService → EventsGateway |
| Badge generation requested | Puppeteer render + R2 upload | RegistrationsService → BadgeGenerationService |
| Badge printed | Print count increment | RegistrationsService |
| Bulk status change | Batch emails (approved/rejected) | RegistrationsService → EmailService |

### 6.2 Public Registration (DUPLICATION)

| Trigger | Side-effect | Service actuel |
|---------|-------------|----------------|
| Public registration | Email confirmation (via `setImmediate`) | PublicService → EmailService |

> ⚠️ **PublicService duplique la logique de `RegistrationsService.create()`** mais sans les WebSocket push ni le badge generation. C'est un candidat à la consolidation.

### 6.3 Print Queue Lifecycle

| Trigger | Side-effect | Service actuel |
|---------|-------------|----------------|
| Print job created | Badge re-generation | PrintQueueService → BadgeGenerationService |
| Print job created | WebSocket push `print-job:created` (ciblé) | PrintQueueService → EventsGateway |
| Print job status updated | WebSocket push `print-job:updated` | PrintQueueService → EventsGateway |
| EMS Print Client connect | Printers registered in-memory | PrintQueueService |
| EMS Print Client disconnect | Printers removed + WS push | PrintQueueService → EventsGateway |
| Printers changed | WebSocket push `printers:updated` | PrintQueueService → EventsGateway |

### 6.4 Invitation Lifecycle

| Trigger | Side-effect | Service actuel |
|---------|-------------|----------------|
| Invitation created | Email invitation | InvitationsService → EmailService |
| Invitation resent | Email invitation | InvitationsService → EmailService |
| Invitation completed | User creation + org membership + role | InvitationsService (transaction) |

### 6.5 Organization Lifecycle

| Trigger | Side-effect | Service actuel |
|---------|-------------|----------------|
| Organization created | Seed 5 default roles + permissions RBAC | OrganizationsService (interne) |

### 6.6 Event Lifecycle

| Trigger | Side-effect | Service actuel |
|---------|-------------|----------------|
| Event created | Tags creation | EventsService → TagsService |

---

## 7. Recommandations pour la migration Event-Driven

### 7.1 Événements domain à définir (priorité)

```typescript
// Registration events
'registration.created'        // → email confirmation + WS push
'registration.approved'       // → email approbation + WS push
'registration.rejected'       // → email refus + WS push
'registration.status_changed' // → WS push
'registration.checked_in'     // → auto table assign + WS push
'registration.checked_out'    // → WS push
'registration.updated'        // → WS push
'registration.deleted'        // → WS push
'registration.badge_printed'  // → print count increment

// Print events
'print_job.created'           // → badge generation + WS push to EMS client
'print_job.status_changed'    // → WS push

// Invitation events
'invitation.created'          // → email invitation
'invitation.resent'           // → email invitation
'invitation.completed'        // → user creation + org membership

// Organization events
'organization.created'        // → seed roles + permissions

// Event lifecycle
'event.created'               // → tags creation + WS push
'event.updated'               // → WS push
'event.deleted'               // → WS push
```

### 7.2 Services à refactorer (par ordre de complexité)

1. **RegistrationsService** (3 138 LOC) — Le plus critique. 4 services injectés, 17+ side-effects. Candidate à un split : `RegistrationCoreService` (CRUD) + event handlers séparés.
2. **PrintQueueService** (280 LOC) — 2 services injectés, state in-memory. Relativement simple.
3. **PublicService** (417 LOC) — Doit être consolidé pour utiliser `RegistrationsService` + émettre le même événement `registration.created`.
4. **InvitationsService** (467 LOC) — 1 service injecté (EmailService). Simple.
5. **EventsService** (1 476 LOC) — 1 service injecté (TagsService). Simple.
6. **BadgeGenerationService** (2 089 LOC) — 2 services injectés mais ce sont des dépendances techniques (storage, templates), pas des side-effects métier. Pas prioritaire.

### 7.3 Quick wins

- **Installer `@nestjs/event-emitter`** et `EventEmitterModule.forRoot()` dans `AppModule`
- **Migrer les WebSocket pushes** dans des handlers `@OnEvent()` dédiés (1 classe par namespace d'événement)
- **Migrer l'envoi d'emails** dans des handlers `@OnEvent()` dédiés
- **Consolider PublicService** pour émettre `registration.created` au lieu de dupliquer la logique
- Les services leaf (Attendees, Users, Roles, etc.) n'ont pas besoin de migration

---

## 8. Métriques récapitulatives

| Métrique | Valeur |
|----------|--------|
| Nombre total de services | 20 (+ 5 utilitaires) |
| LOC total dans les services | **~14 320** |
| Service le plus gros | RegistrationsService (3 138 LOC) |
| Side-effects identifiés | **~25** actions |
| Appels WebSocket directs | **12** emits dans les services |
| Appels Email directs | **7** envois dans 3 services |
| Services fortement couplés | **3** (Registrations, PrintQueue, Invitations) |
| Services sans side-effect | **12** (leaf CRUD services) |
| `@nestjs/event-emitter` installé ? | **Non** |
| Patterns event existants ? | **Aucun** (couplage direct) |
| WebSocket Gateway events | **16** types d'événements émis, **1** reçu |
| Print system state | In-memory `Map` (non persisté) |
