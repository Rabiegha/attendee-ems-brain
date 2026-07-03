# 13 — Information gaps

| # | Gap | Status | Priority | Why it matters | How to answer | Related files | Impact |
|---|---|---|---|---|---|---|---|
| 1 | `ImportBatch` + `ImportRowError` absents | NOT_FOUND | BLOCKING | Impossible de migrer `bulkImport` en BullMQ proprement, ni de fournir rapport d'erreurs persistant | Décider du modèle DB + migration Prisma | [schema.prisma](../../../../../attendee-ems-back/prisma/schema.prisma), [registrations.service.ts](../../../../../attendee-ems-back/src/modules/registrations/registrations.service.ts) L1430+ | P1 imports bloqué |
| 2 | `EmailLog` / `EmailBatch` absents | NOT_FOUND | BLOCKING | Aucune trace des envois, pas de retry, pas de SLA support | Créer modèles + listener post-send | [email.service.ts](../../../../../attendee-ems-back/src/modules/email/email.service.ts) | P1 emails bloqué |
| 3 | `AuditLog` global absent | NOT_FOUND | IMPORTANT | Support et conformité difficiles | Décider scope (qui, quoi, conservation) | tous services | observabilité |
| 4 | État `exposedPrinters` en mémoire | FOUND_IN_CODE | BLOCKING | Incompatible scale horizontal et workers séparés | Déplacer DB ou Redis avec TTL | [print-queue.service.ts](../../../../../attendee-ems-back/src/print-queue/print-queue.service.ts) | P1 prints + scaling |
| 5 | Pas de Socket.IO Redis adapter | INFERRED_FROM_CODE | IMPORTANT | Bloque scale-out > 1 instance backend | Ajouter `@socket.io/redis-adapter` | [websocket.module.ts](../../../../../attendee-ems-back/src/websocket/websocket.module.ts) | GCP multi-instance |
| 6 | Bull Board sans Guard NestJS | FOUND_IN_CODE | IMPORTANT | Risque exposition prod | Guard SUPER_ADMIN + Nginx basic-auth | [bull-board.module.ts](../../../../../attendee-ems-back/src/infra/queue/bull-board.module.ts) | sécurité prod |
| 7 | Pas d'`EventEmitter` / event bus interne | NOT_FOUND | IMPORTANT | Aucun « domain event » découplé ; risque d'utiliser BullMQ comme event bus | Décider d'introduire `@nestjs/event-emitter` pour les domain events | tous services | compat future event-driven |
| 8 | Pas de log JSON structuré (Pino) | INFERRED_FROM_CODE | IMPORTANT | Cloud Logging moins exploitable | Migrer vers `nestjs-pino` ou format custom | [main.ts](../../../../../attendee-ems-back/src/main.ts), middleware logger | obs |
| 9 | Pas de correlation id HTTP↔Job | NOT_FOUND | IMPORTANT | Difficile de tracer une requête utilisateur jusqu'au job | Propager `X-Request-Id` + stocker dans payload BullMQ | partout | obs |
| 10 | Pas de tests sur processors exports | NOT_FOUND (à confirmer) | IMPORTANT | Régressions silencieuses lors d'extension BullMQ | Ajouter `*.processor.spec.ts` | [exports/](../../../../../attendee-ems-back/src/modules/exports) | qualité |
| 11 | Pas de timeout explicite sur SMTP / fetch externes | INFERRED_FROM_CODE | IMPORTANT | Risque blocage thread API | Ajouter timeouts | [email.service.ts](../../../../../attendee-ems-back/src/modules/email/email.service.ts), `n8n-webhook.service.ts` | stabilité |
| 12 | God-service `RegistrationsService` (4113 l., 8 deps) | FOUND_IN_CODE | IMPORTANT | Aggravation garantie si on ajoute `@InjectQueue` sans refactor | Sous-services par responsabilité (`registrations-import`, `registrations-bulk-actions`, `registrations-checkin`) | [registrations.service.ts](../../../../../attendee-ems-back/src/modules/registrations/registrations.service.ts) | dette technique |
| 13 | God-service `BadgeGenerationService` (2228 l.) | FOUND_IN_CODE | IMPORTANT | Couplage Puppeteer + R2 + métier | Séparer `BadgeRenderer` (Puppeteer) + `BadgePersistor` | [badge-generation.service.ts](../../../../../attendee-ems-back/src/modules/badge-generation/badge-generation.service.ts) | P1 badges bulk |
| 14 | Endpoints exports legacy (registrations/attendees `bulkExport`) | FOUND_IN_CODE | NICE_TO_HAVE | Redondance avec module Exports BullMQ | Confirmer dépréciation, migrer front | controllers concernés | propreté |
| 15 | Fichiers source d'import non conservés | FOUND_IN_CODE | IMPORTANT | Pas de replay d'import possible | Stocker source dans R2 + champ `ImportBatch.source_file_key` | [bulkImport](../../../../../attendee-ems-back/src/modules/registrations/registrations.service.ts#L1430) | P1 imports |
| 16 | Rétention `ExportJob.file_url` indéfinie | INFERRED_FROM_CODE | IMPORTANT | Coût R2 + RGPD | Cron BullMQ repeatable purge 30 j | [exports.service.ts](../../../../../attendee-ems-back/src/modules/exports/exports.service.ts) | RGPD + coût |
| 17 | Volume max prod inconnu | NEEDS_HUMAN_ANSWER | BLOCKING | Sizing tout | Question Q1 | — | sizing |
| 18 | Choix Cloud Run vs autre pour workers inconnu | NEEDS_HUMAN_ANSWER | BLOCKING | Architecture déploiement | Question Q2/Q3 | — | infra |
| 19 | Stockage GCS vs R2 indéterminé | NEEDS_HUMAN_ANSWER | IMPORTANT | Refacto `R2Service` éventuelle | Question Q4 | [r2.service.ts](../../../../../attendee-ems-back/src/infra/storage/r2.service.ts) | code |
| 20 | Auth Print Client incertaine | NEEDS_HUMAN_ANSWER | BLOCKING | Sécurité multi-tenant | Question Q5 | [websocket.gateway.ts](../../../../../attendee-ems-back/src/websocket/websocket.gateway.ts), [print-queue.controller.ts](../../../../../attendee-ems-back/src/print-queue/print-queue.controller.ts) | sécurité prod |
| 21 | Comportement multi-org du WS | NEEDS_HUMAN_ANSWER | IMPORTANT | Isolation tenant | Question Q6 | gateway | sécurité |
| 22 | Strategie retry `PrintJob` | NEEDS_HUMAN_ANSWER | IMPORTANT | UX prints | Question Q10 | print-queue | P1 prints |
| 23 | Webhook n8n queue ou non | NEEDS_HUMAN_ANSWER | IMPORTANT | Pertes silencieuses aujourd'hui | Question Q16 | [n8n-webhook.service.ts](../../../../../attendee-ems-back/src/modules/n8n/n8n-webhook.service.ts) | fiabilité |
| 24 | Compatibilité future event-driven désirée ou non | NEEDS_HUMAN_ANSWER | BLOCKING | Conventions Palier 1 | Question Q12 | tous | architecture |
| 25 | Convention naming jobs vs domain events | NOT_FOUND | IMPORTANT | Risque mélange | Définir maintenant (`<domain>.<action>` pour jobs, `<entity>.<past-tense>` pour events) | [queue-names.ts](../../../../../attendee-ems-back/src/infra/queue/queue-names.ts) | gouvernance |

## Synthèse par criticité

- **BLOCKING** (5) : Q1 volume, Q2/Q3 workers/Cloud Run, Q5 auth Print Client, Q12 compat event-driven, + #1 `ImportBatch` + #2 `EmailLog` + #4 `exposedPrinters`.
- **IMPORTANT** (≈14) : refacto god-services, observabilité, naming, etc.
- **NICE_TO_HAVE** (2) : legacy exports, Bull Board UX.
