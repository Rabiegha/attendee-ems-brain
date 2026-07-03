# Workstream — Async architecture

Ce workstream couvre l'identification, la priorisation et la migration des opérations actuellement synchrones (in-process) vers une exécution asynchrone (BullMQ → futur event bus).

## Entrée principale

➡️ [QandA — Discovery & décisions](./QandA/README.md)

Le dossier `QandA/` contient l'analyse complète :

- `01-code-discovery.md` → `14-future-event-driven-compatibility.md`
- `04-async-candidates.md` — catalogue des opérations candidates (sections A/B/C/D)
- `10-gcp-deployment-assumptions.md` — hypothèses d'infrastructure
- `13-information-gaps.md` — questions ouvertes
- `14-future-event-driven-compatibility.md` — compatibilité future event-driven

## Liens connexes

- [Control Center](../../00-CONTROL-CENTER.md)
- [Rabbit hole — full event-driven migration](../../rabbit-holes/full-event-driven-migration.md)
- [Migration GCP](../../other/migrations%20and%20refactoring/migration-GCP/)
