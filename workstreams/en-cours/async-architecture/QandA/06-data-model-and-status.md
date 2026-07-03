# 06 — Modèle de données & suivi de statut

Source : [prisma/schema.prisma](../../../../prisma/schema.prisma) (1217 lignes), [prisma/migrations/](../../../../prisma/migrations/).

## Tableau récapitulatif des modèles « jobs / suivi »

| Modèle attendu | Présent | Référence | Champs clés |
|---|---|---|---|
| `ExportJob` | ✅ `FOUND_IN_CODE` | schema.prisma L1179 | `id, org_id, user_id, type, status, format, columns(Json), filters(Json), email, file_url, file_key, file_size, row_count, eta_seconds, progress(0-100), started_at, completed_at, error_message` |
| `PrintJob` | ✅ `FOUND_IN_CODE` | L1097 | `id, registration_id, event_id, user_id, badge_url, printer_name, status (PENDING/PRINTING/COMPLETED/FAILED/OFFLINE), error, duration_ms, created_at, updated_at` |
| `Badge` (équivaut partiellement à `BadgeGenerationJob`) | ✅ `FOUND_IN_CODE` | L816 | `id, registration_id (unique), badge_template_id, badge_data(Json), data_snapshot(Json), html_snapshot, css_snapshot, qr_code_url, image_url, pdf_url, status (pending/generating/completed/failed), error_message, generated_at, generated_by, print_count, last_printed_at` |
| `BadgeGenerationJob` (modèle dédié file) | ❌ `NOT_FOUND` | — | À créer si on veut découpler générations multiples par registration |
| `ImportBatch` | ❌ `NOT_FOUND` | — | **BLOCKING pour migrer imports en BullMQ** |
| `ImportRowError` | ❌ `NOT_FOUND` | — | **BLOCKING** : aucun moyen aujourd'hui de retourner un rapport d'erreurs partielles téléchargeable |
| `EmailBatch` | ❌ `NOT_FOUND` | — | À créer pour emails en masse |
| `EmailLog` | ❌ `NOT_FOUND` | — | Aucune trace des envois (problème observabilité) |
| `AuditLog` | ❌ `NOT_FOUND` | — | Substituts partiels : `AttendeeRevision`, `BadgePrint`, soft-deletes |
| `Notification` | ❌ `NOT_FOUND` | — | Notifications via WebSocket éphémère uniquement |
| `Job` (générique) | ❌ `NOT_FOUND` | — | Non recommandé — préférer modèle par domaine |

## Champs « statut / progress / retry » présents (grep schema)

| Modèle | Champ | Type | Sens |
|---|---|---|---|
| Invitation | `status` | enum `InvitationStatus` (PENDING/ACCEPTED/EXPIRED/CANCELLED) | flow invit |
| Event | `status` | enum `EventStatus` (draft/published/completed/registration_closed/cancelled/postponed) | lifecycle event |
| Registration | `status` | enum `RegistrationStatus` (awaiting/approved/refused/cancelled) | workflow inscription |
| Badge | `status` | enum `BadgeStatus` (pending/generating/completed/failed) | génération |
| Badge | `error_message`, `generated_at`, `last_printed_at`, `print_count` | — | suivi |
| PrintJob | `status` | enum `PrintJobStatus` | impression |
| PrintJob | `error`, `duration_ms` | — | observabilité |
| ExportJob | `status` | enum `ExportJobStatus` (pending/running/completed/failed) | job |
| ExportJob | `progress` Int | 0-100 | progression |
| ExportJob | `started_at`, `completed_at`, `error_message`, `eta_seconds` | — | observabilité |
| ExportJob | `file_url`, `file_key`, `file_size`, `row_count` | — | livrable |

➡️ **Aucun champ `retryCount`, `attempts`, `failedAt`, `lastError`, `queue` n'existe au niveau Prisma.** Le retry et le compteur d'attempts sont **uniquement gérés côté BullMQ/Redis**. Conséquence : après expiration de la rétention BullMQ (`removeOnFail: 5000`), on perd l'historique des tentatives.

## Source de vérité actuelle par flux

| Flux | Source DB | Source mémoire/Redis | Source WS | Problème |
|---|---|---|---|---|
| Export | ✅ `ExportJob` | Redis (BullMQ progress, attempts) | non | OK |
| Print job | ✅ `PrintJob` | `exposedPrinters` Map (état imprimantes Print Client) | dispatch + ack via WS | État imprimantes perdu au reboot, multi-instance impossible |
| Badge | ✅ `Badge` | non | non | OK |
| Import registrations | ❌ aucune | mémoire pendant la requête HTTP | par-ligne WS (probable) | Crash = perte totale, aucun rapport persistant |
| Email | ❌ aucune | — | — | Aucune trace d'envoi |
| Webhook n8n | ❌ aucune | — | — | Catch-and-log, perte silencieuse |
| Recalc scoring/affinity | partiel (`Attendee.score`, `AttendeeTagAffinity`) | — | — | Recalc en chaîne, aucun job/trace |

## Ce qui devra être persisté avant BullMQ

Pour chaque action que l'on souhaite migrer, **créer la table métier avant de créer la queue** :

1. **`ImportBatch`** : `id, org_id, event_id, user_id, source_file_key, source_file_name, file_size, total_rows, processed_rows, success_count, error_count, status (pending/running/completed/failed/partial), options (Json autoApprove/replaceExisting), error_report_key, started_at, completed_at, error_message`.
2. **`ImportRowError`** : `id, import_batch_id, row_number, raw_row (Json), error_code, error_message, created_at`.
3. **`EmailLog`** (au minimum) : `id, org_id, template_id, recipient, subject, status (queued/sent/failed/bounced), provider_response, retried_count, error_message, related_entity_type, related_entity_id, sent_at, created_at`.
4. **`EmailBatch`** (optionnel mais utile pour les bulk) : `id, org_id, user_id, template_id, total, sent, failed, status, started_at, completed_at`.
5. **`AuditLog`** (recommandé) : `id, org_id, actor_user_id, action (string stable), entity_type, entity_id, payload (Json minimal), ip, user_agent, created_at` — indispensable pour traçer prints, exports, imports, check-ins en cas de support.

> Si on souhaite plus tard un **outbox pattern**, `AuditLog` ou un `OutboxEvent` dédié sera la base. Voir [14-future-event-driven-compatibility.md](14-future-event-driven-compatibility.md).

## Multi-tenancy DB — `FOUND_IN_CODE`

- `org_id String @db.Uuid` indexé sur tous les modèles métier (Attendee, Event, Registration, Badge, BadgeTemplate, EmailTemplate, EmailSender, EventSetting, PartnerScan, Tag, PrintJob, **ExportJob**, etc.).
- Indices composites fréquents : `@@index([org_id, event_id])`, `@@index([org_id, attendee_id])`.
- `Organization.id` → FK + `onDelete: Cascade` (vérifier en migrations si on veut RESTRICT pour les jobs).
- **Tout nouveau modèle de job DOIT porter `org_id` indexé** pour permettre filtrage Bull Board / reprise par tenant.

## Enums existants pertinents

`PrintJobStatus`, `ExportJobStatus`, `ExportJobType` (attendees/registrations), `ExportFormat` (csv/xlsx), `BadgeStatus`, `RegistrationStatus`, `RegistrationSource` (public_form/test_form/manual/import/mobile_app).

➡️ `RegistrationSource.import` existe déjà — utile pour traçer les inscriptions issues d'un import (mais aucune FK vers un `ImportBatch` à ce jour).

## Pourquoi Redis ne doit pas devenir source de vérité métier

1. Rétention BullMQ : `removeOnComplete: 1000`, `removeOnFail: 5000` ⇒ l'historique disparaît.
2. Pas de schéma fort, pas de migrations.
3. Pas de requêtes ad hoc / reporting / support.
4. Pas de scoping `org_id` natif (multi-tenant).
5. Reset / flush Redis = perte. Une panne Memorystore = perte de visibilité.
6. RTO/RPO incohérents entre Postgres (backups Cloud SQL) et Redis.

**Règle d'or à graver pour la suite** :
> Tout job métier a **(a)** une ligne DB créée **avant** l'enqueue, **(b)** `jobId = id DB`, **(c)** statuts mis à jour en DB **dans** le processor, **(d)** payload BullMQ minimal (juste les IDs).
