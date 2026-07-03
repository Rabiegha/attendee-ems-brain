# 11 — Observabilité & tests

## 1. Sentry — `FOUND_IN_CODE`

- Fichier : [src/instrument.ts](../../../../src/instrument.ts) (importé en **premier** dans `main.ts` avant tout autre import).
- Initialisation **conditionnelle** : seulement si `@sentry/node` est installé **et** `SENTRY_DSN` est défini.
- Environnement : `SENTRY_ENVIRONMENT` ou fallback `NODE_ENV`.
- Sampling :
  - Traces : 10 % (`tracesSampleRate: 0.1`).
  - Profiling : 10 % (`profilesSampleRate: 0.1`).
- Integrations : `nodeProfilingIntegration()` (`@sentry/profiling-node`).
- `defaultIntegrations: false` (évite que Sentry n'intercepte le body parser) + `skipOpenTelemetrySetup: true` + `sendDefaultPii: false`.
- Wrapper : [src/common/logger/sentry-logger.ts](../../../../src/common/logger/sentry-logger.ts) (`SentryLogger.info/warn/error/debug`).

## 2. Logger structuré — `INFERRED_FROM_CODE`

- Utilisation du `Logger` NestJS dans la majorité des services (`new Logger(loggerContext)` dans `BaseProcessor`).
- Format JSON structuré : **non** explicite (sortie texte par défaut NestJS). À évaluer si on veut Cloud Logging propre (Pino JSON ou wrapper).
- `RequestLoggerMiddleware` ([src/common/middleware/request-logger.middleware.ts](../../../../src/common/middleware/request-logger.middleware.ts)) appliqué sur toutes les routes (cf. `AppModule.configure`).

## 3. Logs spécifiques jobs

- `BaseProcessor` logge `Job ${id} started/completed/failed`, durée, attempts, erreur, stack.
- Hooks `onSuccess`/`onFailure` log les erreurs silencieuses.
- ⚠️ Pas de **correlation id** entre la requête HTTP qui enqueue et le job (job id BullMQ = id DB, mais pas de trace id Sentry partagé).

## 4. Tests existants — `FOUND_IN_CODE`

- Config : [package.json](../../../../package.json) (`test`, `test:watch`, `test:e2e`, `test:cov`).
- Dossier : [test/](../../../../test/) — config jest-e2e.
- Suites repérées : par convention `*.spec.ts` au niveau modules (à compter par `find src -name "*.spec.ts" | wc -l` — non exécuté ici).
- E2E : config présente mais étendue à vérifier.
- Pas de tests visibles spécifiques aux processors BullMQ (`exports.processor.spec.ts` ? à vérifier).

## 5. Monitoring existant

- Pas de dashboard custom de jobs **applicatif** au-delà de **Bull Board** (`/admin/queues`).
- Pas de `prom-client` / Prometheus metrics dans `package.json`.
- Pas de Grafana / Datadog visible.

## 6. Gaps pour BullMQ

| Gap | Importance |
|---|---|
| Bull Board accessible non protégé (production) | 🔴 |
| Pas de log JSON structuré (limite Cloud Logging) | 🟠 |
| Pas de correlation id HTTP↔Job (trace de bout en bout) | 🟠 |
| Pas de métrique « age oldest job », « queue size », « failure rate » exportée | 🟠 |
| Pas de mécanisme de **replay manuel** d'un job échoué hors Bull Board UI | 🟠 |
| Pas d'alerting (Sentry alerts mais limité aux exceptions) | 🟠 |
| Aucun test sur les processors actuels (couverture inconnue) | 🟠 |
| Pas de test d'intégration end-to-end « enqueue → process → DB update → email envoyé » | 🟠 |
| Pas de `EmailLog` ⇒ impossible de prouver l'envoi a posteriori | 🔴 |

## 7. Comment suivre un jobId de bout en bout aujourd'hui

1. Frontend reçoit `ExportJob.id` à la création (`POST /exports/attendees`).
2. Frontend poll `GET /exports/:id` → champs `status`, `progress`, `file_url`, `error_message`.
3. En backend, logs `BaseProcessor` mentionnent `Job ${job.id}`. Comme `job.id = ExportJob.id`, on peut grep Cloud Logging par cet id.
4. Sentry capture les erreurs avec contexte (mais pas systématiquement `org_id` / `job_id`).
5. Bull Board affiche le job, ses tentatives, ses données.

**Faiblesses** :
- Pas de timeline DB des tentatives (`removeOnFail: 5000` peut purger).
- Pas de log/UI pour le support client « est-ce que l'email d'export est parti à `x@y.com` ? » sans `EmailLog`.

## 8. Ce qui manque pour diagnostiquer les jobs `failed`

- `ExportJob.error_message` est tronqué / sans stack.
- Pas de table `JobAttempt` (historisation des tentatives en DB).
- Pas de bouton « replay » dans le frontend (uniquement via Bull Board ops).

## 9. Ce qui manque pour retry manuellement

- Côté UI client : aucun. Le user qui voit `status: 'failed'` n'a pas de bouton « rejouer » — seul l'admin via Bull Board peut.
- Côté API : pas d'endpoint `POST /exports/:id/retry`.

## 10. Tests nécessaires avant migration

| Niveau | Tests à ajouter avant d'étendre BullMQ |
|---|---|
| Unit | Tests dédiés à chaque méthode lourde de `RegistrationsService` (au minimum `bulkImport` happy path + N erreur partielles). |
| Unit | Tests `BadgeGenerationService.generateBadge` avec mock Puppeteer. |
| Integration | `ExportAttendeesProcessor` : enqueue → processor → ExportJob mis à jour → R2 uploadé (mock R2). |
| Integration | `PrintQueueService.addJob` → PrintJob DB + émission WS observable. |
| E2E | Scénario complet export : `POST /exports/attendees` → poll `GET /exports/:id` → status 'completed' avec `file_url`. |
| Charge | `bulkImport` avec 1k, 5k, 10k lignes (avant migration async). |
| Charge | `generateBadgesForEvent` avec 100, 500 badges (puppeteer sequential vs concurrent). |

## 11. Observabilité minimum à viser avant Palier 1

1. **Log JSON** structuré avec `org_id`, `user_id`, `correlation_id`, `job_id` à chaque ligne.
2. **Bull Board protégé** (Guard SUPER_ADMIN ou basic-auth Nginx).
3. **`EmailLog`** créé.
4. **Endpoint admin replay** d'un job (`POST /jobs/:type/:id/retry`).
5. **Alertes Sentry** sur taux d'échec > X % par queue / heure.
6. **Métriques** : nombre de jobs `failed` / dernière 1h par queue (au minimum exposé sur `/health/jobs` ou Bull Board).
