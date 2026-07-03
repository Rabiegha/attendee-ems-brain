# Rabbit Hole — Massive refactoring

Type: Rabbit Hole
Status: Watchlist
Risk level: High

## 1. Why it is tempting

- Plusieurs **god-services** identifiés (`RegistrationsService` ~4113 l., `BadgeGenerationService` ~2228 l., `PrintQueueService` stateful).
- Plusieurs **duplications** (exports legacy vs module `Exports`).
- Le QandA Async Architecture les liste explicitement ([../async-architecture/QandA/05-coupling-and-side-effects.md](../../workstreams/en-cours/async-architecture/QandA/05-coupling-and-side-effects.md), [../async-architecture/QandA/13-information-gaps.md](../../workstreams/en-cours/async-architecture/QandA/13-information-gaps.md)).
- Tentation de « faire le grand nettoyage » d'un coup pour repartir sur de bonnes bases.

## 2. Why it is dangerous now

- Refactor massif = **risque de régression massif**.
- La couverture de tests est faible / incertaine (cf. [../async-architecture/QandA/11-observability-testing.md](../../workstreams/en-cours/async-architecture/QandA/11-observability-testing.md)) — pas de filet.
- Plusieurs refactors **dépendent de décisions humaines** encore ouvertes (Q1, Q2, Q10 du QandA).
- Un refactor en parallèle de la cartographie Async Architecture **brouille les pistes** : on ne sait plus si une régression vient du refactor ou du chantier en cours.
- Touche multi-modules → review massive, merge conflicts, deploy à risque.

## 3. Scope creep risk

Très élevé :
- « Splitter `RegistrationsService` » dérive vers « repenser les modules Events / Sessions / Attendees ».
- « Nettoyer les exports legacy » dérive vers « migrer le front de tous les usages d'export ».
- « Sortir Puppeteer en worker » dérive vers « créer un nouveau service Docker, nouveau Dockerfile, nouveau pipeline CI/CD ».
- Aucun livrable utilisateur visible pendant des semaines.

## 4. Current decision

- **Refactor au compte-gouttes uniquement.**
- Chaque refactor est **scopé à une fonctionnalité en cours** (ex : pour ajouter une queue `imports.processRegistrationFile`, on extrait `bulkImport` dans son propre service — mais pas plus).
- Pas de PR « refacto » globale.
- Cible des refactors documentée dans [../backlog/refactoring-cleanup.md](../../backlog/a-faire/refactoring-cleanup.md).

## 5. When to revisit

- À chaque chantier (Async Architecture, Onboarding, Billing) qui **nécessite** localement un refactor → faire ce refactor, **scopé**.
- Quand la couverture de tests atteint un niveau qui sécurise un refactor plus large (pas avant).
- Jamais comme initiative « refacto » isolée.

## 6. Related docs

- [../async-architecture/QandA/05-coupling-and-side-effects.md](../../workstreams/en-cours/async-architecture/QandA/05-coupling-and-side-effects.md)
- [../async-architecture/QandA/13-information-gaps.md](../../workstreams/en-cours/async-architecture/QandA/13-information-gaps.md)
- [../async-architecture/QandA/11-observability-testing.md](../../workstreams/en-cours/async-architecture/QandA/11-observability-testing.md)
- [../backlog/refactoring-cleanup.md](../../backlog/a-faire/refactoring-cleanup.md)

## 7. Rule for Copilot

- Si une proposition contient « refactorer X complètement », « splitter le god-service », « grand nettoyage », « PR refacto » → **stop**, pointer vers ce document.
- Autorisé : refactor **local et scopé** lié à une fonctionnalité en cours, avec ticket et tests.
- Interdit : PR multi-fichiers qui touche plusieurs modules « pour faire propre ».
- Interdit : supprimer du code legacy (endpoints, méthodes) sans s'assurer qu'aucun consommateur (front, mobile, print client, n8n) n'en dépend.
