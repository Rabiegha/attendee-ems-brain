# Etat chantier T au 14/07

## Objectif chantier

Mettre un filet de securite test sur les parcours event-critical pour eviter une regression invisible en prod pendant l'evenement.

## Ce qui a ete fait

- T1: `health.e2e-spec.ts`
  - verification `/health`
  - verification `/health/queues`
- T2: `scan-qr.e2e-spec.ts`
  - scan signe valide
  - re-scan deja check-in -> 409 `ALREADY_CHECKED_IN`
  - token falsifie -> 400
  - UUID brut -> 404
  - external_id -> OK
  - cross-event -> 404
  - sans auth -> 401
- T3: `permissions-event.e2e-spec.ts`
  - matrice de permissions sur `/registrations/scan`

## Integrations PR

- PR #18: T1 + T2
- PR #19: T3
- Etat: merge sur `staging`

## Ce qui reste a faire sur T

- P1
  - T4: PDF/badges
  - T5: auth critique KO
  - T6: queue email (BullMQ)
- P2 (si temps)
  - T7: compteur capacite Redis
  - T8: smoke Playwright scan

## Points d'apprentissage T

- les E2E API supertest + Postgres/Redis reels sont deja des tests d'integration utiles
- le contrat `ALREADY_CHECKED_IN` en 409 doit rester stable (important pour le mobile offline)
- un test event-critical doit couvrir les erreurs metier explicites, pas juste les success paths
