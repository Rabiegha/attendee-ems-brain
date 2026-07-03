# Backlog — Refactoring / Cleanup

Type: Backlog Item
Initiative: Stabilization
Workstream: Refactoring / Cleanup
Status: Backlog
Priority: Medium

## 1. Why this exists

Le code back contient plusieurs **god-services** et duplications qui aggraveront la dette technique si on continue d'ajouter des fonctionnalités dessus :

- `RegistrationsService` (~4113 lignes, 8 dépendances) — surtout `bulkImport` (~700 lignes inline).
- `BadgeGenerationService` (~2228 lignes) — couple Puppeteer + R2 + métier.
- `PrintQueueService` — stateful en mémoire (`exposedPrinters: Map`), bloque le scale-out.
- Endpoints d'export legacy (`registrations.bulkExport`, `attendees.bulkExport`) redondants avec le module `Exports` BullMQ.
- Endpoints `/storage/upload/test` et `/storage/badge/generate/:id` exposés sans guard fort.

## 2. Current understanding

- Ces sujets ont été cartographiés en détail dans les Q&A Async Architecture :
  - [../async-architecture/QandA/05-coupling-and-side-effects.md](../../workstreams/a-faire/async-architecture/QandA/05-coupling-and-side-effects.md)
  - [../async-architecture/QandA/13-information-gaps.md](../../workstreams/a-faire/async-architecture/QandA/13-information-gaps.md)
- Aucun refactor n'a été lancé.
- Plusieurs refactors sont **prérequis** à l'extension de BullMQ (imports, badges, prints) — donc ils émergeront naturellement dans le workstream Async Architecture **au compte-gouttes**.

## 3. What is already decided

- **Ne pas** refactorer en masse (cf. rabbit hole [../rabbit-holes/massive-refactoring.md](../../rabbit-holes/a-faire/massive-refactoring.md)).
- Les refactors prérequis à BullMQ sont **scopés** par chantier (un service à la fois).
- Les endpoints `/storage/...` de test ne sont pas supprimés sans ticket explicite.
- Bug handling rule : pas de fix automatique sans ticket (voir [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#bug-handling-rule)).

## 4. What is not decided yet

- Ordre des refactors (RegistrationsService.bulkImport en premier ? BadgeGenerationService ?).
- Stratégie pour `PrintQueueService` (déplacer l'état mémoire → DB ? Redis ?).
- Sort des endpoints exports legacy (déprécier ? supprimer ? migrer le front d'abord ?).
- Couverture de tests cible avant chaque refactor.
- Limite de taille de service (au-delà de N lignes, refactor obligatoire ?).

## 5. Why not now

- Focus actuel = cartographie / Q&A Async Architecture.
- Refactorer sans tests et sans contrat clair = risque de régression.
- Plusieurs refactors dépendent de décisions humaines en attente (Q1, Q2, Q10 du QandA).

## 6. When to revisit

- **Au coup par coup**, quand un refactor devient prérequis d'un chantier ciblé (ex : ajouter une queue `imports.processRegistrationFile` exigera de scinder `bulkImport`).
- Jamais en mode « grand nettoyage » global.

## 7. Related docs

- [../async-architecture/QandA/05-coupling-and-side-effects.md](../../workstreams/a-faire/async-architecture/QandA/05-coupling-and-side-effects.md)
- [../async-architecture/QandA/13-information-gaps.md](../../workstreams/a-faire/async-architecture/QandA/13-information-gaps.md)
- [../async-architecture/QandA/08-import-export-file-processing.md](../../workstreams/a-faire/async-architecture/QandA/08-import-export-file-processing.md)
- [../async-architecture/QandA/07-websocket-print-client.md](../../workstreams/a-faire/async-architecture/QandA/07-websocket-print-client.md)
- [../rabbit-holes/massive-refactoring.md](../../rabbit-holes/a-faire/massive-refactoring.md)

## 8. Possible future epics

- **Split RegistrationsService** : `registrations-import.service`, `registrations-bulk-actions.service`, `registrations-checkin.service`.
- **Split BadgeGenerationService** : `BadgeRenderer` (Puppeteer) + `BadgePersistor` (R2 + DB).
- **PrintQueueService stateless** : déplacer `exposedPrinters` en DB ou Redis (TTL).
- **Sunset legacy exports** : déprécier `registrations.bulkExport` / `attendees.bulkExport` après migration front.
- **Storage test endpoints cleanup** : guarder ou supprimer `/storage/upload/test`, `/storage/badge/generate/:id`.

## 9. Notes for Copilot

- **Ne pas refactorer** un god-service entier en une fois.
- **Ne pas supprimer** d'endpoint legacy sans ticket.
- Tout refactor doit s'accompagner de tests sur la zone touchée.
- Si un refactor est nécessaire pour avancer sur Async Architecture, le **scope strictement** à la fonctionnalité en cours.
