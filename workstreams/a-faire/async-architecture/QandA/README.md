# QandA — Phase de découverte (avant proposition d'architecture async)

> Périmètre : **backend NestJS** (`attendee-ems-back/`). Quelques références au front, mobile et print client uniquement quand nécessaire.

## 1. But de cette phase

Cette phase **ne** propose **aucune** architecture finale, **aucun** plan de migration, **aucun** code BullMQ.
Elle sert uniquement à :

1. Documenter ce que le code existant fait **réellement**.
2. Identifier les **vrais** candidats à l'asynchrone.
3. Mesurer la **dette technique** et le **couplage** actuels avant d'introduire (ou d'étendre) BullMQ/Redis.
4. Vérifier que les futurs choix ne **bloqueront pas** une éventuelle évolution vers une architecture event-driven (Pub/Sub GCP, outbox, etc.).
5. Lister les **questions humaines** réellement bloquantes — pas plus.

## 2. Pourquoi cette étape avant BullMQ/Redis

> **Découverte clé** : BullMQ, `@nestjs/bullmq`, ioredis, Bull Board et un `BaseProcessor` **existent déjà** dans le code. Voir [04-async-candidates.md](04-async-candidates.md) et [05-coupling-and-side-effects.md](05-coupling-and-side-effects.md).

Le sujet n'est donc pas « introduire BullMQ », mais « **généraliser proprement** un usage déjà partiellement présent ». La dette risquerait d'empirer si :

- on enfonçait la logique métier dans des processors (perte de testabilité, perte d'audit DB) ;
- on faisait de Redis la source de vérité métier (badges, prints, imports) ;
- on conservait les god-services actuels (registrations 4113 lignes, events 2447, badges 2228) en y empilant des `@InjectQueue` ;
- on émettait WebSocket/Email/Webhook directement depuis partout (déjà le cas) au lieu de centraliser sur un listener post-job.

## 3. Comment lire les fichiers

| Fichier | Contenu |
|---|---|
| [01-code-discovery.md](01-code-discovery.md) | Fichiers/dossiers inspectés, patterns recherchés, résumé brut |
| [02-modules-map.md](02-modules-map.md) | Cartographie des modules backend |
| [03-current-flows.md](03-current-flows.md) | Flux métier (print, import, export, email, check-in, scan, badges…) |
| [04-async-candidates.md](04-async-candidates.md) | Classement sync vs async par flux |
| [05-coupling-and-side-effects.md](05-coupling-and-side-effects.md) | **Le plus important** : couplage, dette, risques d'aggravation |
| [06-data-model-and-status.md](06-data-model-and-status.md) | Modèles Prisma de statuts/jobs et ce qui manque |
| [07-websocket-print-client.md](07-websocket-print-client.md) | WebSocket + Print Client Electron |
| [08-import-export-file-processing.md](08-import-export-file-processing.md) | Imports XLSX, exports CSV/XLSX, fichiers générés |
| [09-security-multitenancy.md](09-security-multitenancy.md) | Guards, permissions, orgId, risques cross-tenant |
| [10-gcp-deployment-assumptions.md](10-gcp-deployment-assumptions.md) | Hypothèses GCP (Cloud Run, Memorystore, Cloud SQL) |
| [11-observability-testing.md](11-observability-testing.md) | Sentry, logs, tests, gaps |
| [12-open-questions-for-human.md](12-open-questions-for-human.md) | ≤ 20 questions humaines |
| [13-information-gaps.md](13-information-gaps.md) | Tableau récapitulatif des gaps |
| [14-future-event-driven-compatibility.md](14-future-event-driven-compatibility.md) | Hors scope mais critique : compatibilité future event-driven |

## 4. Statuts d'information

Chaque affirmation est étiquetée :

| Statut | Sens |
|---|---|
| `FOUND_IN_CODE` | Information vue directement dans un fichier (chemin cité) |
| `INFERRED_FROM_CODE` | Déduction raisonnable, plusieurs indices convergents |
| `NOT_FOUND` | Cherché et non trouvé dans le code accessible |
| `NEEDS_HUMAN_ANSWER` | Décision produit/équipe que le code ne peut pas résoudre |
| `BLOCKING` | Bloque l'audit final ou le plan de migration |

## 5. Étape suivante

Une fois ces fichiers relus et les questions humaines de [12-open-questions-for-human.md](12-open-questions-for-human.md) traitées (au minimum les `BLOCKING`), on pourra :

1. Produire l'audit final dans `docs/async-architecture/AUDIT.md`.
2. Décider du périmètre du Palier 1 (probablement : exports déjà faits, prints à formaliser, imports à migrer).
3. Définir les conventions (naming queues, payload minimal, idempotence, transaction boundaries, domain events).
4. Rédiger un plan de migration progressif.

**Ne pas écrire de code BullMQ supplémentaire avant cette étape.**
