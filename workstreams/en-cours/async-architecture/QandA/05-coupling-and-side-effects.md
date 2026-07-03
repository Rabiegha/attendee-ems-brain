# 05 — Couplage & effets secondaires (fichier critique)

> Objectif : déterminer si introduire/étendre BullMQ aggraverait la dette technique existante ou s'inscrirait dans une trajectoire saine.

## 1. God-services identifiés — `FOUND_IN_CODE`

| Service | Lignes | Méthodes publiques | Dépendances injectées |
|---|---|---|---|
| `RegistrationsService` | **4113** | ~31 | `PrismaService`, `BadgeGenerationService`, `EventsGateway`, `EmailService`, `EventTablesService`, `AttendeeScoringService`, `AttendeeAffinityService`, `N8nWebhookService` (**8 deps**) |
| `EventsService` | **2447** | ~27 | `PrismaService`, `TagsService` (forwardRef), `AttendeeScoringService`, `AttendeeAffinityService` |
| `BadgeGenerationService` | **2228** | ~16 | `PrismaService`, `R2Service`, `BadgeTemplatesService` (+ Puppeteer singleton) |
| `AttendeesService` | **1533** | ~12 | `PrismaService`, `AttendeeScoringService` |
| `PrintQueueService` | 280 | ~12 | `PrismaService`, `BadgeGenerationService`, `EventsGateway` |
| `ExportsService` | 295 | ~6 | `PrismaService`, `EmailService`, `R2Service`, 2 × `Queue` |

➡️ **Verdict** : 4 services dépassent largement les seuils raisonnables (1000+ lignes). Ajouter des `@InjectQueue` à `RegistrationsService` ou `BadgeGenerationService` **sans refactor préalable empirerait** la dette.

## 2. Mélange logique métier ↔ effets secondaires — `FOUND_IN_CODE`

### `RegistrationsService` — points concrets

| Méthode | Effets secondaires en ligne | Référence approx. |
|---|---|---|
| `create()` (L854-1095) | DB transaction + capacity check + `EmailService.sendFromCustomTemplate()` + `eventsGateway.emitRegistrationCreated()` | L1007, L1046, L1060 |
| `bulkImport()` (L1430-2131, **700 lignes**) | Parse XLSX + validation + N transactions + (probable) émissions WS par ligne + emails conditionnels | L1813 (tx), L20-21 (xlsx) |
| `bulkUpdateStatus()` (L3699-3890) | Boucle DB + envoi emails sync + emissions WS | — |
| `checkIn()` (L3075-3240) | DB + recalc scoring/affinity + WS `emitRegistrationCheckedIn` + `n8nWebhookService.notifyCheckIn()` | L3218, L3221, L3336 |
| `approveWithEmail` / `rejectWithEmail` | DB + Email sync + WS | L668, L736, L819 |

➡️ **Pattern problématique** : la même méthode fait DB + Email + WS + Webhook. Si on remplace `EmailService.sendEmail()` par `emailsQueue.add()` sans découpler, on **enfonce** la dette : la responsabilité de « publier les conséquences » reste dans le service métier.

### `BadgeGenerationService`

- Couplage **fort** à Puppeteer (singleton `onModuleInit`, `launch()` au boot).
- Couplage **fort** à R2 (upload direct dans la méthode de génération).
- Pas de séparation `render → persist → notify`.

### `PrintQueueService`

- État des imprimantes en `Map<orgId, Map<deviceId, printers[]>>` **en mémoire** ⇒ perdu au reboot, **incompatible multi-instance horizontale**.
- Émet directement sur `EventsGateway` (couplage transport ↔ métier).
- Callback de disconnect enregistré côté `EventsGateway` puis exécuté par `PrintQueueService` — couplage circulaire implicite.

### `InvitationsService`

- Injection directe de `EmailService`. Si SMTP tombe, l'invitation est créée mais l'email part en erreur HTTP — pas de retry.

### `EventsService`

- Génère ExcelJS in-memory dans le contrôleur. Pour de gros events ⇒ OOM possible.

## 3. Logique métier dans controllers — `INFERRED_FROM_CODE`

- D'après l'examen des controllers vus (`registrations`, `events`, `attendees`), les controllers délèguent globalement au service (pas de god-controller flagrant). C'est cohérent NestJS.
- Quelques endpoints transversaux sont placés sur `EventsController` alors qu'ils délèguent à `RegistrationsService` (`bulk-import`, `generate-badges`). Pas dramatique, mais flou de responsabilité (cf. [12-open-questions-for-human.md](12-open-questions-for-human.md)).

## 4. Duplication / redondance détectées

- **Exports** : coexistence `module Exports` (BullMQ, `ExportJob`) **et** `registrations.bulkExport()` / `attendees.bulkExport()` (sync ExcelJS). Probable migration partielle non finalisée.
- **Génération de badges** : `RegistrationsService.generateBadgesForEvent/generateBadgesBulk` orchestre vers `BadgeGenerationService.generateBulk` + `BadgeGenerationService.generateEventBadgesPDF` — au moins 3 entrées pour des intentions proches.

## 5. Transaction boundaries

| Endroit | `prisma.$transaction` | Risque |
|---|---|---|
| `registrations.create()` L859 | Upsert attendee + create registration + sync table/session | ✅ correct |
| `registrations.bulkImport()` L1813 | **Une tx par ligne dans une boucle** | ❌ N transactions, pas de progress, pas de reprise |
| `events.create()` L93 | Event + settings | ✅ correct |
| `attendees.create()` (~L37) | Attendee + revision | ✅ correct |

➡️ **Aucune transaction n'englobe une émission WS ou un envoi email**. C'est juste — mais cela signifie aussi que **le commit DB et la notification ne sont pas atomiques**. Une erreur SMTP après commit ⇒ état incohérent silencieux. C'est exactement ce qu'un **outbox pattern** résoudrait plus tard (cf. [14-future-event-driven-compatibility.md](14-future-event-driven-compatibility.md)).

## 6. Effets secondaires par dépendance

| Dépendance | Appelants | Risque actuel |
|---|---|---|
| `EmailService` | `RegistrationsService`, `InvitationsService`, `ExportsService`, processors Exports | Tous sync, pas de retry, pas de log d'envoi |
| `EventsGateway` | `RegistrationsService`, `PrintQueueService` (et probablement `EventsService`, `SessionsService`) | Émissions inline ; pas de garantie ordre/livraison ; risque de spam pendant bulk |
| `BadgeGenerationService` | `RegistrationsService`, `PrintQueueService` | Puppeteer bloquant ; pas d'isolation worker |
| `R2Service` | `BadgeGenerationService`, `ExportsService` processors, `StorageController` | OK ; pas d'abstraction au-dessus pour swap (Cloud Storage GCP) |
| `N8nWebhookService` | `RegistrationsService.checkIn` | Fire-and-forget catch-and-log, perte silencieuse possible |
| `AttendeeScoringService` / `AttendeeAffinityService` | `RegistrationsService`, `AttendeesService`, `PartnerScansService`, `EventsService`, `SessionsService` (indirect) | Recalc en chaîne, potentiellement coûteux ; aucune queue |

## 7. EventEmitter / bus interne — `NOT_FOUND`

- Aucun `EventEmitter2`, `@OnEvent`, `eventEmitter.emit`. **L'app n'a pas d'event bus** : tout passe par appel direct entre services (couplage explicite).
- Conséquence : aucun « domain event » découplé n'existe aujourd'hui. Le risque, en introduisant BullMQ partout, est de prendre BullMQ pour un event bus — ce qui serait un anti-pattern (cf. [14-future-event-driven-compatibility.md](14-future-event-driven-compatibility.md)).

## 8. Audit / observabilité métier — `NOT_FOUND`

- Pas de `AuditLog` model. Substituts partiels : `AttendeeRevision`, `BadgePrint`, soft-deletes (`deleted_at`).
- Pas de trace persistante des emails envoyés, des webhooks n8n appelés, des dispatchs WS.
- ⇒ Difficulté à **prouver** qu'un email/print/webhook a bien eu lieu (BLOCKING pour SLA, support client).

## 9. Risques d'aggravation si on ajoute BullMQ sans refactor

| Choix | Risque |
|---|---|
| Ajouter `@InjectQueue(EMAILS)` dans `RegistrationsService` | God-service devient pire (9ᵉ dépendance) |
| Mettre toute la logique d'import dans un processor | On déplace la dette de la méthode de 700 lignes vers le processor ; aucun gain de testabilité |
| Émettre WebSocket depuis le processor d'export | Couplage transport↔processor, double émission possible (job retry) |
| Conserver l'état `exposedPrinters` en mémoire | Incompatible avec workers séparés et scaling horizontal |
| Stocker la progression d'import dans Redis seul | Redis devient source de vérité métier (anti-pattern) |
| Lancer Puppeteer dans le process API | Conflit mémoire avec API ; doit aller dans worker dédié |

## 10. Verdict global

> **Architecture acceptable mais à encadrer**, avec **3 zones à refacto avant d'étendre BullMQ** :
>
> 1. **`RegistrationsService.bulkImport`** : doit être extrait dans un sous-service (`registrations-import.service`) avec étapes `parse → validate → persist → notify` séparées, **avant** d'introduire une queue d'import.
> 2. **`BadgeGenerationService`** : doit séparer `render` (Puppeteer) de `persist` (R2 + Badge) ; et exposer un point d'entrée job-friendly (`generateBadge(registrationId)` ré-entrant et idempotent).
> 3. **`PrintQueueService`** : doit déplacer l'état `exposedPrinters` en DB (ou Redis avec TTL et conscience explicite que c'est de la coordination, pas du métier) avant tout worker séparé.
>
> Pour les autres (exports event-level, emails transactionnels, webhook n8n), la migration peut suivre le pattern **déjà éprouvé** par le module `Exports`.

### Zones à risque élevé

- 🔴 **`bulkImport` registrations** (700 lignes, O(N) tx)
- 🔴 **Print Queue stateful in-memory** (incompatible scale-out)
- 🔴 **Pas d'`AuditLog` / `EmailLog`** (observabilité métier insuffisante)
- 🟠 Couplage `RegistrationsService` (8 deps)
- 🟠 Puppeteer dans le process API
- 🟠 Émissions WebSocket inline pendant bulk operations

### Zones saines / à répliquer

- 🟢 Module `Exports` (pattern DB-first + BullMQ + R2 stream + email final)
- 🟢 `BaseProcessor` (lifecycle logging, hooks `onSuccess`/`onFailure`)
- 🟢 Multi-tenancy `org_id` consistant et indexé
- 🟢 Validation `class-validator` + DTO + Zod sur la config
- 🟢 Sentry conditionnel, body-parser-safe
