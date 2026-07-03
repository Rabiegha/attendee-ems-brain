# 08 — Imports / Exports / File processing

## A. Imports

### A.1 Import registrations XLSX — `FOUND_IN_CODE`

- **Endpoint** : `POST /events/:eventId/registrations/bulk-import` ([src/modules/events/events.controller.ts](../../../../src/modules/events/events.controller.ts) ~L340-370)
- **Multipart** : `FileInterceptor('file')`
- **Query params** : `autoApprove` (boolean), `replaceExisting` (boolean)
- **Service** : `RegistrationsService.bulkImport()` ([src/modules/registrations/registrations.service.ts](../../../../src/modules/registrations/registrations.service.ts) L1430-2131, **~700 lignes**)
- **Librairies** : `xlsx` (parsing) — `import * as XLSX from 'xlsx'` (L20)
- **Pipeline (en mémoire)** : `XLSX.read(buffer)` → `XLSX.utils.sheet_to_json()` → boucle sur les lignes → par ligne : validation + `prisma.$transaction` (upsert attendee + create/update registration + table/session choices).
- **Mapping colonnes** : géré par alias dans la méthode (first_name/firstname/prenom…). Pas de configuration externe.
- **Validation** : inline dans la boucle. Pas de schema Zod dédié à l'import.
- **Erreurs partielles** : ⚠️ `NOT_FOUND` — aucune table `ImportRowError`. Le service collecte probablement des erreurs en mémoire et les retourne dans la réponse HTTP, mais elles **ne sont pas persistées** ⇒ impossibilité de retrouver un rapport d'import a posteriori.
- **Limites actuelles** :
  - **Sync** : si > N lignes (ordre 5–10k selon perf), timeout HTTP probable (`bodyParser` 50 MB depuis `main.ts`, mais pas de timeout explicite).
  - **Mémoire** : tout le fichier est chargé en RAM via `XLSX.read(buffer)`. Pas de streaming.
  - **Transactions** : 1 transaction par ligne. Connection pool sous tension pour gros imports.
- **Async readiness** : ❌ **Aucune**. Pas de queue, pas de modèle `ImportBatch`, pas de `progress`, pas de retry, pas de reprise.
- **Risques** : timeout passerelle (Nginx, Cloud Run), OOM, perte du résultat si client ferme l'onglet.

### A.2 Template d'import

- `GET /registrations/template?lang={lang}` → `RegistrationsService.generateTemplate(lang)` (sync ExcelJS, OK).
- Template par event (pour récupérer les champs de formulaire de l'event) : `generateTemplateForEvent()` (sync).

### A.3 Autres imports

- ❌ Pas d'import attendees autonome (les attendees sont créés via l'import registrations).
- ❌ Pas d'import companies/tags/badge-templates.
- ❌ Scripts de migration historiques dans `scripts/migration-data-from-ems-to-attendee/` — exécution one-shot CLI, hors flux applicatif.

## B. Exports

### B.1 Module `Exports` — async BullMQ — `FOUND_IN_CODE`

- Chemins : [src/modules/exports/](../../../../src/modules/exports/)
- **Endpoints** :
  - `POST /exports/attendees/estimate` — estimation
  - `POST /exports/attendees` — création job
  - `POST /exports/registrations/estimate`
  - `POST /exports/registrations`
  - `GET /exports/:id` — poll statut
  - `PATCH /exports/:id/notify` — attacher email après coup
- **Formats** : `csv` (via `csv-stringify`) ou `xlsx` (via `exceljs`).
- **Streaming** : `PassThrough` → `R2Service.uploadStream()` (multipart). Pas de buffer global.
- **Livraison** : `file_url` = `R2Service.getSignedDownloadUrl(file_key, 72h)`. Email optionnel via `ExportReadyEmailTemplate` (attachement si fichier ≤ 18 MB sinon lien).
- **Modèle DB** : `ExportJob` (cf. [06-data-model-and-status.md](06-data-model-and-status.md)).
- **Idempotence** : `jobId = ExportJob.id` (DB d'abord, enqueue ensuite).
- **Retry** : `attempts: 3`, backoff exponentiel 5 s (depuis `DEFAULT_JOB_OPTIONS`). `onFailure` met `status: 'failed'` après épuisement.
- **Where builders** : `attendee-export-where.ts` (287 l.), `registration-export-where.ts` (121 l.) — extraction propre des filtres réutilisables.
- **Confiance** : élevée. Pattern à répliquer.

### B.2 Exports events synchrones — `FOUND_IN_CODE`

- `GET /events/:id/export-stats` → `EventsService.exportEventStats()` (~300 lignes, multi-sheets ExcelJS).
- `GET /events/:id/export-sessions` → `exportEventSessions()` (~195 lignes, multi-sheets).
- `GET /events/:id/export-placements` → `exportEventPlacements()` (~115 lignes, multi-sheets).
- `POST /events/bulk-export` → `bulkExport()` (~60 lignes, multi-events CSV/XLSX).
- **Tech** : `new ExcelJS.Workbook()` → `writeBuffer()` → response.
- **Risque** : sync, in-memory, **timeout / OOM** pour gros events. À migrer en BullMQ via `ExportJob.type` étendu (`event-stats`, `event-sessions`, `event-placements`, `events-bulk`).

### B.3 Exports « legacy » coexistants

- `POST /registrations/bulk-export` → `RegistrationsService.bulkExport()` (registrations.service.ts L2367-2492, sync ExcelJS).
- `POST /attendees/bulk-export` → `AttendeesService.bulkExport()` (sync ExcelJS).
- ⚠️ Redondance avec module `Exports` BullMQ. À clarifier (`NEEDS_HUMAN_ANSWER` — Q9).

### B.4 Exports partner-scans

- `GET /partner-scans/export` (CSV), `GET /partner-scans/export-excel` (XLSX). Sync.

## C. Génération de fichiers (badges)

- Badge unitaire : `BadgeGenerationService.generateBadge()` → Puppeteer PDF → R2 (`uploadBadgePdf`).
- Badge bulk : `generateBulk()` → boucle séquentielle.
- PDF multi-pages event : `generateEventBadgesPDF()` (badge-generation.service.ts L1460-1767).
- Image badge (screenshot) : `getBadgeImageBuffer()` (L2094+).
- Tech : `puppeteer@24.29.1`, Chromium fallback via `PUPPETEER_EXECUTABLE_PATH`.

## D. Stockage fichiers — `FOUND_IN_CODE`

- Provider : **Cloudflare R2** (S3-compatible) via `@aws-sdk/client-s3` + `@aws-sdk/lib-storage` (multipart).
- Bucket : `R2_BUCKET_NAME` (défaut `ems-badges`).
- Keys utilisés :
  - `badges/{registrationId}/badge.pdf`
  - `badges/{registrationId}/badge.{ext}`
  - `test/{timestamp}-{filename}`
  - exports : key construit dans le processor (probable `exports/{org_id}/{job_id}.{format}` — à vérifier ligne précise).
- Endpoints publics :
  - `POST /storage/upload/test` — test upload (à protéger / supprimer en prod).
  - `GET /storage/badge/generate/:id` — génération badge test.
  - `GET /storage/signed-url/:key` — presigned upload 1 h.
  - `GET /storage/debug/config` — diagnostic.
- Download : pas d'endpoint de proxy ; on utilise directement le `file_url` signé R2 (72 h pour les exports).

## E. Upload côté front

- `multer` via `FileInterceptor` — pas d'upload direct R2 depuis le front aujourd'hui (sauf via presigned URLs).
- Limite body : 50 MB (`main.ts` après Sentry).

## F. Progress et reprise

| Action | Progress | Reprise |
|---|---|---|
| Export attendees / registrations | ✅ `exportsQueue.updateProgress()` + `ExportJob.progress` (0-100) | ✅ via retry BullMQ (3 attempts) |
| Export events stats/sessions/placements | ❌ | ❌ |
| Bulk-export attendees/registrations legacy | ❌ | ❌ |
| Import XLSX | ❌ | ❌ (transaction par ligne ⇒ pas de reprise au milieu) |
| Génération badges bulk | ❌ | ❌ |

## G. Risques transverses

- **Timeout passerelle** Nginx / Cloud Run : par défaut 60 s pour Cloud Run HTTP. Tous les exports sync des sections B.2-B.4 sont à risque.
- **OOM** : ExcelJS in-memory pour gros events.
- **Perte fichier source d'import** : le fichier XLSX uploadé n'est **pas conservé** ⇒ impossible de rejouer un import.
- **R2 → migration Cloud Storage GCP** : R2 est S3-compatible, le swap est techniquement possible mais le SDK `@aws-sdk/client-s3` resterait pertinent même contre Cloud Storage via HMAC keys (alternative : `@google-cloud/storage`). Décision humaine.

## H. Async readiness par flux

| Flux | Async ready | Action minimale |
|---|---|---|
| Export attendees | ✅ | rien |
| Export registrations | ✅ | rien |
| Export events stats/sessions/placements | ❌ | étendre `ExportJob.type`, créer 3 nouveaux processors |
| Export partner-scans | ❌ | étendre `ExportJob.type` |
| Import XLSX | ❌ | créer `ImportBatch`, `ImportRowError`, persister fichier source, refactor `bulkImport` |
| Génération badges bulk | ❌ | créer queue dédiée + worker isolé Puppeteer |
| Emails bulk | ❌ | créer `EmailBatch` + queue |
