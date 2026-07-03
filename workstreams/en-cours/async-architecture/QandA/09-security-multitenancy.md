# 09 — Sécurité & multi-tenancy

## 1. Guards en place — `FOUND_IN_CODE`

- `JwtAuthGuard` ([src/auth/](../../../../src/auth/)) — vérifie le JWT access token.
- `RequirePermissionGuard` / `PermissionsGuard` ([src/common/guards/](../../../../src/common/guards/)) — vérifie les permissions RBAC.
- `ApiKeyGuard` — utilisé sur n8n (`X-API-Key`).
- `ThrottlerGuard` (`@nestjs/throttler`) — protection rate-limit.
- Module `AuthzModule` (hexagonal, étape 3) — couche `platform/authz/` centralise la logique RBAC + CASL (`@casl/ability` dans `package.json`).

## 2. Décorateurs / pattern d'utilisation

- `@UseGuards(JwtAuthGuard, RequirePermissionGuard)` au niveau classe sur la plupart des controllers métier.
- `@RequirePermission('permission.code')` au niveau méthode.
- `@CurrentUser()`, `@CurrentOrgId()` (probable) — extraction du contexte depuis le JWT.

## 3. Cartographie des routes async / bulk / print / import / export

| Endpoint | Guard / Permission attendue | Confiance |
|---|---|---|
| `POST /events/:eventId/registrations/bulk-import` | JwtAuth + Permission `registrations.import` ou équivalent | À vérifier explicitement — `INFERRED_FROM_CODE` |
| `POST /registrations/bulk-checkin` | Permission `registrations.checkin` ou `bulk` | `INFERRED_FROM_CODE` |
| `PATCH /registrations/bulk-update-status` | Permission `registrations.update` | `INFERRED_FROM_CODE` |
| `DELETE /registrations/bulk-delete` | Permission `registrations.delete` | `INFERRED_FROM_CODE` |
| `POST /registrations/bulk-export` | Permission `registrations.read` / `export` | `INFERRED_FROM_CODE` |
| `GET /registrations/template` | Permission `registrations.read` | `INFERRED_FROM_CODE` |
| `POST /events/bulk-export` | Permission `events.read` / `export` | `INFERRED_FROM_CODE` |
| `GET /events/:id/export-stats` | Permission `events.read` | `INFERRED_FROM_CODE` |
| `GET /events/:id/export-sessions` | Permission `events.read` | `INFERRED_FROM_CODE` |
| `GET /events/:id/export-placements` | Permission `events.read` | `INFERRED_FROM_CODE` |
| `POST /exports/attendees` | JwtAuth + Permission `exports.create` (ou `attendees.export`) | `INFERRED_FROM_CODE` (controller imports AuthzModule) |
| `POST /exports/registrations` | Idem | `INFERRED_FROM_CODE` |
| `GET /exports/:id` | Permission `exports.read` | `INFERRED_FROM_CODE` |
| `POST /print-queue/add` | JwtAuth + permission print | À vérifier explicitement |
| `POST /print-queue/printers` | À vérifier — appelé par Print Client (JWT du client ?) | **NEEDS_HUMAN_ANSWER** Q5 |
| `GET /print-queue/client-status` | Lecture admin | À vérifier |
| `PATCH /print-queue/:id/printing` `/complete` `/fail` | Print Client (avec JWT ?) | À vérifier |
| `POST /attendees/bulk-export` | Permission `attendees.export` | `INFERRED_FROM_CODE` |
| `POST /attendees/bulk-delete` | Permission `attendees.delete` | `INFERRED_FROM_CODE` |
| `GET /partner-scans/export` | Permission `partner_scans.read` | `INFERRED_FROM_CODE` |
| `POST /storage/upload/test` | À vérifier — risque si exposé public | **NEEDS_HUMAN_ANSWER** Q4 |
| `GET /storage/signed-url/:key` | À vérifier (presigned upload) | **NEEDS_HUMAN_ANSWER** |
| `/admin/queues/*` (Bull Board) | ⚠️ **Pas de Guard NestJS** (commentaire dans `bull-board.module.ts`) | **FOUND_IN_CODE** — risque si exposé public |
| `/n8n/*` | `ApiKeyGuard` + `ThrottlerGuard` | `FOUND_IN_CODE` |
| `/health` | Public | `FOUND_IN_CODE` |

➡️ **Routes async/bulk/print/import/export sans protection identifiée** :
- 🔴 **Bull Board** (`/admin/queues`) — pas de Guard NestJS. **Doit être restreint par Nginx / VPN / basic-auth en prod**. Code lui-même note l'absence et recommande un Guard SUPER_ADMIN.
- ⚠️ **`/storage/upload/test`** et **`/storage/badge/generate/:id`** — endpoints de test, à supprimer ou guarder strictement en prod.

## 4. Multi-tenancy — `FOUND_IN_CODE`

- Source de vérité `orgId` :
  - **JWT access token** porte `userId` + `orgId` actif (ou `currentOrgId`) — décoration `@CurrentOrgId()` (probable).
  - Un utilisateur peut être membre de plusieurs orgs via `OrgUser` (composite `[user_id, org_id]`) et choisir l'org active.
- Validation `eventId/orgId` : pratique courante via `event.org_id === currentOrgId` dans les services. **À vérifier systématiquement dans les méthodes bulk** (cf. risques).
- `assigned scope` : enum `PermissionScope { any, org, assigned, none, own }`. Une permission scopée `assigned` doit vérifier que l'event est dans la liste `event_assigned_users`. ⚠️ Pour les opérations bulk, **assurer que chaque entité touchée respecte la portée**.

## 5. Risques cross-tenant identifiés

| Risque | Source | Mitigation actuelle |
|---|---|---|
| Job exporté avec un `orgId` du payload différent du JWT | Si le processor BullMQ fait confiance au payload sans revérifier | À encadrer : **le processor doit relire `ExportJob` en DB et utiliser `ExportJob.org_id`**, jamais le payload brut |
| Print Client usurpant `orgId` dans `ems-client:identify` | gateway WS | À vérifier (le handler croise-t-il avec `socket.orgId` issu du JWT ?) |
| `printer_name` choisi par le caller au lieu de la liste exposée par le Print Client | `PrintQueueService.addJob` | Vérifier que `printer_name` est validé contre la liste connue pour l'org |
| Endpoint `POST /print-queue/printers` exposé sans guard fort | À vérifier | Recommander JWT obligatoire + match `device.orgId == jwt.orgId` |
| Lien signé R2 partagé entre orgs | URL signée 72 h | Risque limité (URL imprévisible), mais **ne pas inclure de PII dans le path** ; et `ExportJob.file_url` ne doit être lisible que par les membres de l'org propriétaire (`GET /exports/:id` filtre par orgId — à confirmer) |
| Bull Board (`/admin/queues`) public | bull-board.module.ts | À restreindre Nginx |

## 6. Données sensibles dans payloads BullMQ

- Aujourd'hui (exports) : payload = `{ jobId, orgId }`. ✅ Minimal, pas de PII.
- Convention à graver pour la suite : **payload BullMQ minimal = juste les IDs**. Charger les données nécessaires depuis la DB dans le processor avec `org_id` rechargé.

## 7. Recommandations pour les workers (à appliquer, pas encore implémenter)

1. **Le processor doit toujours rechargé l'entité** (`ExportJob`, `ImportBatch`, `PrintJob`) depuis Postgres et utiliser ses champs `org_id`/`event_id`/`user_id`. Le payload Redis ne sert que de pointeur.
2. **Revalider les permissions** dans le processor pour les actions qui agissent sur des données externes (ex. envoyer un email avec PII).
3. **Logger `org_id` + `user_id` + `job_id` + `correlation_id`** dans chaque log de processor (utile pour Sentry et audit).
4. **Aucun secret dans le payload** (tokens, mots de passe, signatures).
5. **Bull Board protégé** par un Guard SUPER_ADMIN ou un middleware Nginx basic-auth dès la mise en prod.

## 8. Tableau par action async candidate

| Action | Permission cible | orgId source | eventId source | userId initiateur | Validation actuelle | Validation à refaire dans le worker | Risque cross-tenant |
|---|---|---|---|---|---|---|---|
| Export attendees | `attendees.export` (à confirmer) | JWT → `ExportJob.org_id` | filters (`event_id`) snapshot | JWT → `ExportJob.user_id` | À l'enqueue | Worker relit ExportJob et filtre WHERE `org_id` | Faible (déjà géré) |
| Export registrations | Idem | Idem | Idem | Idem | Idem | Idem | Faible |
| Export events stats (futur async) | `events.export` | À définir | request param | JWT | À l'enqueue | Worker filtre par `org_id` | Moyen si on oublie |
| Import registrations XLSX (futur async) | `registrations.import` | JWT | URL param `:eventId` | JWT | Sync à l'enqueue | Worker relit `ImportBatch.org_id` + `event_id` | **Élevé** si payload contient le buffer brut |
| Génération badges bulk (futur) | `badges.generate` | JWT | request | JWT | À l'enqueue | Worker filtre `event.org_id` | Moyen |
| Print job (futur queue serveur) | déjà via PATCH endpoints | DB `PrintJob.org_id` | DB | DB | Print Client patche par id | Worker (si on en ajoute) doit relire DB | Moyen (printer_name) |
| Email bulk (futur) | `emails.send` | JWT → `EmailBatch.org_id` | optionnel | JWT | À l'enqueue | Worker filtre `org_id` des recipients | **Élevé** : ne jamais accepter une liste d'emails arbitraire sans vérifier qu'ils appartiennent à l'org |
| Webhook n8n (futur queue) | interne | depuis Registration | depuis Registration | depuis Registration | check-in handler | Worker relit Registration | Faible |
