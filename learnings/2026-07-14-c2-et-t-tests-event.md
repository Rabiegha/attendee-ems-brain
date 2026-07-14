# Learning — 2026-07-14 — C2 + chantier T

## C2 (Mailgun integration)

- Toujours valider le dernier metre sur les appels templatises: une cle metier (`idempotencyKey`) au mauvais niveau peut passer inaperçue en review rapide.
- Le package verification minimal avant PR a bien fonctionne:
  - lint cible
  - typecheck
  - unit
  - e2e
- Le merge C2 a ete fluidifie en isolant le scope dans une seule PR orientee domaine (`email`, `config`, `prisma`).

## T (tests event-critical)

- P0 apporte un gain immediat de confiance prod en couvrant:
  - health real dependencies
  - securite scan QR
  - permissions event
- Le contrat `ALREADY_CHECKED_IN` (409) est un point d'interface important avec le mobile offline: il faut le traiter comme contrat stable.
- Les E2E supertest avec Postgres/Redis reels donnent deja une couche integration suffisamment forte pour l'event.

## Execution

- Quand un chantier est bloque par l'ops (ex: 0-MON VPS), preparer la checklist executable en amont permet de reprendre vite des que les prerequis tombent.
- Tenir un recap de cloture par chantier reduit le temps de reprise le lendemain.
