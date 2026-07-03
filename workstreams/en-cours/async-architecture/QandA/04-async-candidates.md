# 04 — Candidats async

Classement basé sur le code réel (cf. [03-current-flows.md](03-current-flows.md)).

## A. Sync obligatoire (NE PAS migrer en queue)

| Action | Raison | Notes |
|---|---|---|
| Check-in unitaire (`registrations.checkIn`) | UX scanner exige réponse < 200 ms | Side-effects (webhook n8n, email) peuvent être queue-és séparément |
| Scan partenaire (`partner-scans.createScan`) | Idem, action utilisateur | Recalc scoring/affinity reste en sync ou async léger |
| Scan session (`sessions.scanParticipant`) | Idem | OK sync |
| Lecture/CRUD standards | — | OK |
| Génération QR code (`/email/qrcode` endpoints) | Léger | OK |
| Génération HTML/PDF base64 unitaire pour aperçu badge | Léger, < 1 s | OK |
| Templates exports (`/registrations/template`) | < 1 s | OK |

## B. Async recommandé

| Action | Raison actuelle | Bénéfice BullMQ | Priorité | Complexité | Idempotence | Progress | Retry | Table DB |
|---|---|---|---|---|---|---|---|---|
| Export events stats / sessions / placements | Sync ExcelJS, gros events → timeout | Délivrer fichier en lien signé, plus de timeout HTTP | P2 | Moyenne | Job id = `ExportJob.id` | utile | 3 | Étendre `ExportJob.type` |
| `events/bulk-export` (multi-events) | Sync ExcelJS | Idem | P2 | Faible | Idem | utile | 3 | Idem |
| `attendees/bulk-export` & `registrations/bulk-export` (legacy) | Sync ExcelJS | Idem + harmoniser avec module Exports | P2 | Faible | Idem | utile | 3 | Idem |
| `partner-scans/export*` | Sync | Idem | P3 | Faible | Idem | utile | 3 | Idem |
| Webhook n8n check-in | Fire-and-forget HTTP en plein checkIn | Retry, observabilité, ne pas polluer le path UX | P2 | Faible | Clé : `registration_id + scanned_at` | non | 3-5 | optionnel (audit) |
| Emails transactionnels (invitation, reset, approve/reject, confirmation) | Sync Nodemailer dans le request | Retry, rate-limit SMTP, non-blocking UX, tracer envois | P2 | Moyenne (besoin `EmailLog`) | Clé : `template_id + recipient + entity_id` | non | 3-5 | **À créer : `EmailLog`** |
| **Recalc score & affinity unitaire** (`AttendeeScoringService.recalculateScore` + `AttendeeAffinityService.recalculateAffinities`) | Aujourd'hui appelés en **fire-and-forget `.catch(() => {})`** depuis 15+ call sites : `registrations.service.ts` (create, update, status change, bulk-update, bulk-checkin), `partner-scans.service.ts`, `events.service.ts`, `public.service.ts`, `attendee-access.service.ts`, `attendees.service.ts`. Affinity = SQL raw JOIN + LATERAL + DELETE/INSERT en transaction → non négligeable. Risque : perte si process crash, pas de retry, surcharge DB sur opérations bulk | Debounce par `attendee_id` (un seul recalc même si 10 events arrivent), retry, observabilité, pas de path UX impacté | P2 | Faible | Clé : `attendee_id` (le job le plus récent gagne) | non | 3 | Réutiliser pattern queue, pas de table nouvelle requise (le score est sur `Attendee.score`) |

## C. Async obligatoire (sans queue, l'app saute aux limites)

| Action | Raison | Bénéfice | Priorité | Complexité | Idempotence | Progress | Retry | Table DB |
|---|---|---|---|---|---|---|---|---|
| **Import registrations XLSX** (`bulkImport`) | 700 lignes, O(N) transactions, aucun suivi, pas de reprise, pas d'erreurs partielles téléchargeables | Progress, erreurs partielles consultables, reprise, isolation worker | **P1** | **Élevée** (refacto préalable obligatoire) | Hash fichier + `org_id + event_id` | **oui** | 3 | **À créer : `ImportBatch` + `ImportRowError`** |
| **Génération badges en masse** (`registrations.generateBadgesForEvent`, `generateBulk`, `generateEventBadgesPDF`) | Puppeteer séquentiel, bloque le main process | Worker isolé avec Puppeteer dédié, concurrence contrôlée, progress | **P1** | Moyenne | Clé : `registration_id + badge_template_id` | oui | 3 | Étendre `Badge` (`progress` ?), ou créer `BadgeGenerationJob` |
| **Emails en masse** (option `bulkUpdateStatus` + boucle) | SMTP rate-limit, blocage requête, perte d'envois en cas de panne | Throttling, retry, observabilité | **P1** | Moyenne | Clé : `batch_id + recipient` | oui | 3 | **À créer : `EmailBatch` + `EmailLog`** |
| **Print badges** (`PrintQueueService.addJob` en masse) | WebSocket-only, état imprimantes en mémoire, pas de retry serveur, perte si backend redémarre | Persistance dispatch, retry, scheduling, observabilité | **P1** | Moyenne (sans casser WS Print Client) | `print_job_id` | utile | 3 | `PrintJob` existe |
| **Recalc score org-wide** (`AttendeeScoringService.recalculateAllScores`) exposé via [`POST /attendees/recalculate-scores`](../../../../../attendee-ems-back/src/modules/attendees/attendees.controller.ts) | Endpoint admin **synchrone** : balaie tous les attendees de l'org, batchs de 500, requête HTTP bloquée pendant toute la durée. Pour 50 k attendees → minutes voire timeout reverse-proxy. Idem `recalculateAllAffinities` (endpoint sœur) | Réponse 202 immédiate, progress via table, worker isolé | **P1** | Faible | Clé : `org_id` (un seul recalc complet à la fois par org) | **oui** | 1 (recalc complet n'a pas vraiment de sens à retry, mais le batch oui) | **À créer : `ScoreRecalcJob`** (org_id, status, progress, started_at, finished_at, total, updated, error) |
| **Recalc affinity en cascade sur changement de tag** ([`tags.service.ts:158`](../../../../../attendee-ems-back/src/modules/tags/tags.service.ts)) | Boucle fire-and-forget sur **toutes les registrations** rattachées à un tag modifié, chacune déclenchant un `recalculateAffinities` (SQL coûteux). 1 modif de tag sur un gros event = N affinity recalc en parallèle, sans throttling | Enqueue 1 job par attendee avec dédup, throttling concurrence | **P1** | Faible | Clé : `attendee_id` (dédup) | utile | 3 | Pas de table nouvelle, réutilise le job unitaire ci-dessus |

## D. À analyser (NEEDS_HUMAN_ANSWER ou refacto requis)

| Action | Question ouverte |
|---|---|
| WebSocket `emitRegistrationCreated/Updated/CheckedIn` lors de `bulkImport` | Spammer le WS pendant un import de 5000 lignes vs émettre un événement `import.completed` final ? |
| Recalc scoring/affinity après chaque check-in/scan | Garder fire-and-forget actuel, ou queue obligatoire (cf. ligne ajoutée en B) ? Trade-off : sync = score à jour immédiatement dans la réponse / queue = pas de garantie de fraîcheur mais retry et obs |
| Cron `EventAutoCompleteService` (15 min) | Garder ScheduleModule local ou utiliser BullMQ repeatable jobs ? |
| Endpoint `bulk-checkin` | Vraiment async ou maintenir sync pour feedback immédiat ? |

## E. À ne pas migrer

| Action | Raison |
|---|---|
| Endpoints de configuration (organizations, users, roles, permissions, templates) | Aucun coût async |
| Endpoints de lecture (`findAll`, `findOne`) | Doivent rester sync, optimiser côté DB |
| Bull Board UI | Outil ops, hors scope |
| n8n endpoints (lecture seule) | API tierce, sync OK |

---

## Notes transverses

- **Idempotence** : aujourd'hui, la seule garantie est `jobId = ExportJob.id` côté Exports. Toute nouvelle queue doit suivre le pattern « **DB d'abord, enqueue avec id = id DB** ».
- **Progress** : seul `ExportJob.progress` existe. À généraliser sur `ImportBatch`, `BadgeGenerationJob`, `EmailBatch`.
- **Table de suivi** : Redis ne doit jamais être source de vérité métier. Tout job métier doit avoir sa ligne DB (statut, progress, error).
- **Workers séparés vs in-process** : actuellement les processors tournent **dans le même process** que l'API NestJS (cf. `exports.module.ts`). Pour les charges lourdes (Puppeteer, gros imports), un **worker séparé** sera nécessaire. C'est une décision GCP (cf. [10-gcp-deployment-assumptions.md](10-gcp-deployment-assumptions.md)).
