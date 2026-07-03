# Rabbit Hole — Immediate bug fixing

Type: Rabbit Hole
Status: Watchlist
Risk level: Medium-High

## 1. Why it is tempting

- Les bugs documentés dans [../bugs/](../../bugs) sont **visibles**, **détaillés**, **prêts à corriger en apparence**.
- L'instinct développeur dit « corrigeons tant qu'on a le contexte en tête ».
- Copilot peut être tenté, dès qu'il lit un bug review, de proposer un patch immédiat.
- Les analyses contiennent souvent une section « Recommandation » qui ressemble à un plan d'action prêt à coder.

## 2. Why it is dangerous now

- Un **bug review = analyse**, pas un mandat (voir [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#bug-handling-rule)).
- Plusieurs bugs documentés touchent des zones structurantes :
  - [../bugs/cors-origin-security-review.md](../../bugs/a-faire/cors-origin-security-review.md) → touche tous les endpoints, le WebSocket, et la prod immédiatement.
  - [../bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md) → multi-tenant, RGPD, cascade — voir rabbit hole [user-account-deletion-anonymization.md](user-account-deletion-anonymization.md).
- Sans tests dédiés, un fix peut casser plus qu'il ne corrige.
- Sans ticket et sans owner, un fix passe sous le radar (pas de QA, pas de communication).
- Le focus actuel est **Async Architecture** ; corriger des bugs en parallèle disperse l'attention.

## 3. Scope creep risk

Élevé :
- « Corriger un bug » dérive vite vers « refactorer tout le module concerné ».
- Un fix CORS peut entraîner une réécriture de la conf Nginx, du gateway WS, des origins front/mobile/print-client.
- Un fix user deletion peut entraîner un chantier RGPD complet.
- Une fois lancé, plus d'arrêt possible avant déploiement.

## 4. Current decision

- **Pas de correction automatique** des bugs documentés.
- Chaque bug fix exige : (a) un ticket explicite, (b) un scope défini, (c) des tests, (d) un déploiement planifié.
- Les bugs restent **visibles** dans :
  - [../00-CONTROL-CENTER.md](../../../attendee-ems-back/docs/00-CONTROL-CENTER.md) (section 5).
  - [../steering/NOW.md](../../../attendee-ems-back/docs/steering/NOW.md) (section 2).
  - [le backlog](../../backlog/README.md).

## 5. When to revisit

- Bug par bug, **avec ticket explicite** et décision humaine.
- Quand un bug devient bloquant en production.
- Quand le workstream Async Architecture aura libéré du temps.

## 6. Related docs

- [../bugs/](../../bugs)
- [../bugs/cors-origin-security-review.md](../../bugs/a-faire/cors-origin-security-review.md)
- [../bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md)
- [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#bug-handling-rule)
- [user-account-deletion-anonymization.md](user-account-deletion-anonymization.md)

## 7. Rule for Copilot

- Si une proposition consiste à **corriger un bug listé dans `docs/bugs/`** sans ticket explicite → **stop**, pointer vers ce document.
- Autorisé : **compléter** une bug review (impact, options, reproduction, tests à prévoir).
- Autorisé : **ouvrir** un ticket de fix avec scope clair et validation humaine.
- Interdit : éditer le code (controller, service, guard, conf Nginx, gateway WS, schema Prisma) pour patcher un bug sur la simple base de sa documentation.
