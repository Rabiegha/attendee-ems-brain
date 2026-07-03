# Rabbit Hole — Full event-driven migration now

Type: Rabbit Hole
Status: Watchlist
Risk level: High

## 1. Why it is tempting

- Le sujet event-driven est à la mode et structurant.
- Le projet utilise déjà BullMQ → on peut croire qu'il « suffit » de tout transformer en events.
- Permettrait de découpler radicalement les modules métier.
- Préparerait des intégrations externes (n8n, Zapier, CRM, marketing).
- Le QandA Async Architecture évoque déjà le sujet ([../async-architecture/QandA/14-future-event-driven-compatibility.md](../workstreams/async-architecture/QandA/14-future-event-driven-compatibility.md)).

## 2. Why it is dangerous now

- Aucun **event bus** interne en place (`EventEmitter2` absent).
- Aucun `AuditLog` / `OutboxEvent` ⇒ pas de fondation pour la cohérence DB ↔ events.
- Aucune **transaction boundary** qui englobe le commit DB + l'émission d'event.
- Les services métier (`RegistrationsService` ~4113 lignes) sont câblés en dur sur Email / WS / Webhook → migrer en event-driven exigerait un refactor massif simultané.
- BullMQ partiellement utilisé : seulement 2 queues (`exports.attendees`, `exports.registrations`). On n'a même pas encore généralisé les commands → vouloir y ajouter une couche d'events est prématuré.
- Risque de **mélanger** commands et events dans le même transport (BullMQ devient un faux event bus, dette garantie).
- Choix techniques structurants (Pub/Sub GCP ? Kafka ? NATS ?) ouverts → décision irréversible si on s'engage trop tôt.

## 3. Scope creep risk

Très élevé :
- Bus interne → outbox → broker externe → contrats versionnés → tests d'intégration cross-modules.
- Touche tous les services métier en même temps.
- Pousse à réécrire le RBAC, les guards, l'auth pour s'adapter au nouveau pattern.
- Mobilise plusieurs sprints entiers sans livrable utilisateur visible.

## 4. Current decision

- **Pas maintenant.**
- Le travail Async Architecture en cours pose les **conventions compatibles** avec une future migration event-driven (DB-first, jobId = id DB, payload minimal, naming `<domain>.<action>` vs `<entity>.<past_tense>`, idempotence) sans implémenter le bus.
- À court terme : étendre BullMQ aux commands manquantes (imports, badges, emails, prints, webhook n8n), créer `AuditLog` / `EmailLog`, refactorer les god-services au compte-gouttes.

## 5. When to revisit

- Quand **plusieurs intégrations externes** (≥ 3) commenceront à câbler le même flux (check-in, registration) — preuve qu'un bus serait rentable.
- Quand `AuditLog` / `OutboxEvent` sera en place et stable.
- Quand les conventions Palier 1 (cf. [../async-architecture/QandA/14-future-event-driven-compatibility.md](../workstreams/async-architecture/QandA/14-future-event-driven-compatibility.md)) seront actées et respectées partout.

## 6. Related docs

- [../async-architecture/QandA/14-future-event-driven-compatibility.md](../workstreams/async-architecture/QandA/14-future-event-driven-compatibility.md)
- [../async-architecture/QandA/05-coupling-and-side-effects.md](../workstreams/async-architecture/QandA/05-coupling-and-side-effects.md)
- [../async-architecture/QandA/13-information-gaps.md](../workstreams/async-architecture/QandA/13-information-gaps.md)
- [../workstreams/async-architecture/README.md](../workstreams/async-architecture/README.md)

## 7. Rule for Copilot

- Si une proposition contient « migrer en event-driven », « ajouter Pub/Sub », « créer un bus d'events », « émettre des domain events partout » → **stop**, pointer vers ce document.
- Autorisé : poser des **conventions compatibles** event-driven dans le workstream Async Architecture (sans implémenter un bus).
- Autorisé : ajouter `@nestjs/event-emitter` **uniquement** sur un ou deux flux critiques validés (check-in, badge generated) si décidé explicitement.
- Interdit : refactorer tous les services pour émettre des events « pour préparer ».
