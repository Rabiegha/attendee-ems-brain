# 03 — Flux actuels

Légende des colonnes :
- **Sync/Async** : nature actuelle.
- **WS** : émissions WebSocket associées.
- **Email** : envoi d'email associé.
- **Fichier** : production d'un fichier.
- **Statut DB** : modèle Prisma persistant le flux.
- **Retry** : politique de retry visible.
- **Risque migration BullMQ** : si on migrait tel quel sans refactor.

---

## A. Impressions de badges

### A.1 Print d'un badge unitaire — `FOUND_IN_CODE`
- Endpoint : `POST /print-queue/add` ([src/print-queue/print-queue.controller.ts](../../../../../attendee-ems-back/src/print-queue/print-queue.controller.ts))
- Controller : `PrintQueueController.addJob()`
- Service : `PrintQueueService.addJob()`
- Modèles : `PrintJob` (Prisma, [schema.prisma#L1097](../../../../../attendee-ems-back/prisma/schema.prisma))
- Sync/Async : **Hybride** — endpoint sync qui crée un `PrintJob` en DB et **émet via WebSocket** au Print Client connecté.
- WS : oui (`printers:updated`, et probablement push du job au Print Client cible).
- Email : non. Fichier : badge image/PDF généré côté `BadgeGenerationService` (URL stockée dans `Registration.badge_image_url`/`badge_pdf_url`).
- Statut DB : enum `PrintJobStatus` (`PENDING`, `PRINTING`, `COMPLETED`, `FAILED`, `OFFLINE`).
- Retry : `POST /print-queue/offline/retry-all`. Pas de retry automatique en backend (le Print Client retente).
- Failure : `PATCH /print-queue/:id/fail` met à jour le statut + `error` + `duration_ms`.
- Risque migration BullMQ : **MOYEN**. Le flux est déjà event-like (WS), mais l'état des imprimantes est en mémoire (`Map`). BullMQ apporterait : retry serveur, scheduling, observabilité. À condition de **ne pas casser le couplage WS au Print Client**.
- Confiance : **élevée** sur la persistance, **moyenne** sur le flux de dispatch (la méthode `addJob` du service mériterait relecture détaillée pour le call WS exact).

### A.2 Print en masse / génération badges pour tout un event
- Endpoint : `POST /events/:eventId/registrations/generate-badges` et `.../generate-badges-bulk` ([src/modules/events/events.controller.ts](../../../../../attendee-ems-back/src/modules/events/events.controller.ts))
- Controller : `EventsController` → délègue à `RegistrationsService.generateBadgesForEvent()` / `generateBadgesBulk()` (registrations.service.ts, méthodes ~L2628, 2705, 2826) → `BadgeGenerationService.generateBulk()` (badge-generation.service.ts L935–1015).
- Sync/Async : **Sync, séquentiel** par registration (Puppeteer).
- WS : non documenté explicitement pour cette voie. Email : non. Fichier : N PDF uploadés R2.
- Statut DB : `Badge.status` (`pending|generating|completed|failed`) + `error_message`, `print_count`, `last_printed_at`.
- Retry : aucun automatique.
- Risque migration BullMQ : **ÉLEVÉ si pas refacto** — bloque le thread principal, Puppeteer est lourd (mémoire) et nécessite un worker dédié si on veut concurrence + timeouts. Bénéfice énorme à migrer.

---

## B. Génération de badges (PDF/image)

### B.1 Génération d'un badge unitaire
- Endpoint : `POST /events/:eventId/registrations/:id/generate-badge`
- Service : `BadgeGenerationService.generateBadge()` (badge-generation.service.ts L630–935 — 305 lignes)
- Tech : **Puppeteer headless** (singleton, init `onModuleInit()`), HTML template + font auto-shrink, PDF, upload R2 (`uploadBadgePdf`), update `Badge.status`.
- Sync/Async : **Sync**.
- Modèles : `Badge`, `BadgeTemplate`.
- Risque migration : **doit absolument passer async** dès qu'on traite > 1 badge ou que la connexion est lente.

### B.2 Récupération HTML/PDF base64
- `GET /badge-generation/:id/html`, `/pdf-base64`, `/metadata` — lecture pure (sync OK).

---

## C. Imports

### C.1 Import registrations XLSX — `FOUND_IN_CODE`
- Endpoint : `POST /events/:eventId/registrations/bulk-import` (multipart `file`, query `autoApprove`, `replaceExisting`) — [events.controller.ts](../../../../../attendee-ems-back/src/modules/events/events.controller.ts) ~L340-370
- Lib : `xlsx` ([registrations.service.ts L20](../../../../../attendee-ems-back/src/modules/registrations/registrations.service.ts))
- Service : `RegistrationsService.bulkImport()` L1430–2131 (~700 lignes)
- Sync/Async : **Sync. Une transaction Prisma par ligne (O(N))**.
- WS : non confirmé pendant l'import (probablement émissions par-ligne via `emitRegistrationCreated`/`Updated`).
- Email : possible si `autoApprove` déclenche emails d'approbation.
- Fichier : non (juste lecture).
- Statut DB : pas de modèle `ImportBatch` ni `ImportRowError` — **NOT_FOUND**. Aucun suivi persistant des erreurs partielles, aucun rapport d'import téléchargeable.
- Retry : non. Failure : transaction = rollback de la ligne. Pas de reprise possible.
- Risque migration BullMQ : **ÉLEVÉ** car la méthode mélange parsing + validation + IO + transactions. À refactorer **avant** BullMQ pour : (a) découper `parse → validate → persist`, (b) introduire `ImportBatch` + `ImportRowError`, (c) émettre progress.

### C.2 Template d'import
- `GET /registrations/template?lang=` — sync ExcelJS, OK tel quel.
- `GET /registrations/template-for-event/:eventId` (probable) — sync.

---

## D. Exports

### D.1 Export attendees / registrations — DÉJÀ async (BullMQ) — `FOUND_IN_CODE`
- Endpoints : `POST /exports/attendees`, `POST /exports/registrations`, `POST /exports/*/estimate`, `GET /exports/:id`, `PATCH /exports/:id/notify`
- Service : `ExportsService` ([exports.service.ts](../../../../../attendee-ems-back/src/modules/exports/exports.service.ts))
- Processors : `ExportAttendeesProcessor`, `ExportRegistrationsProcessor` extending `BaseProcessor`.
- Modèle : `ExportJob` (schema.prisma L1179) — status, progress 0-100, eta_seconds, file_url, file_key, file_size, row_count, error_message, email.
- Tech : `csv-stringify` ou `exceljs`, streaming `PassThrough` → R2 `uploadStream` (multipart). Email `ExportReadyEmailTemplate` envoyé si `email` renseigné. Lien signé 72 h.
- Retry : `attempts: 3` + backoff exponentiel 5 s (`DEFAULT_JOB_OPTIONS`). `onFailure` marque DB `failed` après épuisement.
- WS : non. Idempotence : `jobId = ExportJob.id` (DB d'abord, enqueue ensuite).
- Risque migration : **N/A**, c'est le modèle de référence. À répliquer pour les autres exports.

### D.2 Exports events (stats, sessions, placements, bulk-export) — sync
- Endpoints : `GET /events/:id/export-stats`, `/export-sessions`, `/export-placements`, `POST /events/bulk-export`
- Service : `EventsService.exportEventStats()` (~300 lignes), `exportEventSessions()` (~195), `exportEventPlacements()` (~115), `bulkExport()` (~60).
- Tech : ExcelJS in-memory `writeBuffer()` → réponse HTTP.
- Sync/Async : **Sync, risque timeout** pour gros events / multi-events.
- Risque migration : **MOYEN** — passer en async BullMQ + `ExportJob.type = 'event-stats' | 'event-sessions' | 'event-placements' | 'events-bulk'`.

### D.3 Export attendees `bulkExport` & registrations `bulkExport` (non BullMQ — code legacy ?)
- `POST /registrations/bulk-export`, `POST /attendees/bulk-export` (sync ExcelJS).
- Coexistent avec le module Exports BullMQ. Probable **redondance/dette**.
- Statut : à clarifier — voir [12-open-questions-for-human.md](12-open-questions-for-human.md).

### D.4 Exports partner-scans
- `GET /partner-scans/export` (CSV), `/export-excel` (XLSX). Sync.

---

## E. Emails

### E.1 Email transactionnel (invitation, reset password, confirmation registration, approval, rejection)
- Service : `EmailService.sendFromCustomTemplate()` / `sendEmail()` (Nodemailer).
- Sync/Async : **Sync**, dans le thread HTTP.
- Retry : aucune file. Si SMTP tombe, l'utilisateur reçoit une erreur 5xx.
- Appelants : `RegistrationsService` (approve/reject/confirm, lignes 736, 819, 1046, 1060), `InvitationsService` (lignes 135, 310), `ExportsService` (notify late), processors d'exports (ExportReady).
- Modèle DB : `EmailTemplate`, `EmailSender`, `EmailSetting` (configuration) — **pas de `EmailLog` ni `EmailBatch`**, aucune trace d'envoi persistée.

### E.2 Email en masse
- **NOT_FOUND** comme endpoint dédié. La voie actuelle est `POST /registrations/bulk-update-status` avec option email (`bulkUpdateStatus()` L3699-3890) qui boucle et envoie sync.
- Risque migration BullMQ : **ÉLEVÉ bénéfice**, surtout pour limiter le rate SMTP et tracer les échecs.

---

## F. Check-in / scan

### F.1 Check-in unitaire — `FOUND_IN_CODE`
- Endpoint : `POST /registrations/:id/check-in` (probable), `POST /registrations/scan`
- Service : `RegistrationsService.checkIn()` L3075-3240, `checkInByCode()`.
- Side effects : `Registration.checked_in_at/by/checkin_location`, recalc scoring/affinity, **WebSocket emit `emitRegistrationCheckedIn`** (L3218, 3336), **webhook n8n** (L3221) `N8nWebhookService.notifyCheckIn()` (fire-and-forget, 5 s timeout, catch-and-log).
- Sync/Async : **Sync** — doit le rester (UX scanner).
- Retry : non nécessaire (action utilisateur).
- Risque migration BullMQ : **NE PAS migrer le check-in lui-même**. En revanche, la **notif n8n** et un éventuel email post-check-in pourraient être queue-able.

### F.2 Bulk check-in
- `POST /registrations/bulk-checkin` → `bulkCheckIn()` (L3887+). Sync.

### F.3 Scan partenaire
- `PartnerScansService.createScan()` + recalc scoring/affinity. Sync.

### F.4 Scan session (présence)
- `POST /sessions/:id/scan` → `SessionsService.scanParticipant()`. Sync, raw SQL pour counts.

---

## G. Sessions / Rapports / Stats

- `getPresentCount()`, `getUniqueVisitorsCount()` : `$queryRaw` distinct. Sync. OK.
- `EventsService.exportEventStats()` : sync ExcelJS, voir D.2.
- Pas de rapport périodique ; deux crons :
  - `EventAutoCompleteService` (`*/15 * * * *`) — transition `published → completed` quand `end_at` passé.
  - `EventPurgeService` (`0 5 * * *`) — purge soft-deleted > 30 jours.

---

## H. Upload / Download fichiers

- `POST /storage/upload/test` (FileInterceptor → R2). Sync.
- `GET /storage/signed-url/:key` — presigned upload (sync, rapide).
- Téléchargement export : via `ExportJob.file_url` (lien signé R2 72 h). Pas d'endpoint backend pour streamer.

---

## I. Bulk operations divers

- Registrations : bulkCheckIn, bulkUpdateStatus, bulkDelete, bulkExport, bulkImport.
- Attendees : bulkDelete, bulkRestore, bulkPermanentDelete, bulkExport.
- Partner-scans : bulkDelete, bulkRestore, bulkPermanentDelete.
- Events : bulkExport.
- Badge-generation : `generateBulk`, `generateEventBadgesPDF`.

Tous **sync** sauf les exports attendees/registrations du module `Exports`.

---

## Résumé matrice

| Flux | Sync/Async | WS | Email | Fichier | Statut DB | Retry | Risque migration |
|---|---|---|---|---|---|---|---|
| Print badge unitaire | Hybride | ✓ | — | PDF/img (R2) | `PrintJob` | manuel | Moyen |
| Print bulk | Sync | partiel | — | PDF (R2) | `Badge` | non | Élevé |
| Génération badge unitaire | Sync | — | — | PDF/img | `Badge` | non | Élevé |
| Génération badges event | Sync séq. | — | — | PDF | `Badge` | non | Élevé |
| Import registrations XLSX | Sync | par-ligne | possible | — | aucun (`ImportBatch` absent) | non | **Très élevé** |
| Export attendees / registrations | **Async BullMQ** | — | optionnel | CSV/XLSX (R2) | `ExportJob` | 3 attempts | N/A — référence |
| Export events stats/sessions/placements | Sync | — | — | XLSX | aucun | non | Moyen |
| Export partner-scans | Sync | — | — | CSV/XLSX | aucun | non | Moyen |
| Email transactionnel | Sync | — | ✓ | — | aucun | non | Moyen-Élevé |
| Email en masse | (via bulk-update-status) | — | ✓ N | — | aucun | non | Élevé |
| Check-in | Sync | ✓ | possible | — | `Registration` | — | Ne pas migrer (sauf side-effects) |
| Scan partenaire | Sync | — | — | — | `PartnerScan` | — | Sync OK |
| Scan session | Sync | — | — | — | `SessionScan` | — | Sync OK |
| Cron auto-complete events | Sync | — | — | — | `Event` | — | OK |
| Cron purge | Sync | — | — | — | — | — | OK |
| Webhook n8n check-in | Sync fire-and-forget | — | — | — | aucun | catch-and-log | À queue-er |
