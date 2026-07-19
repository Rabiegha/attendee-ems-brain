# Sessions de travail autonome — LFD 2026

Ce dossier conserve le contrat de chaque session de travail prolongée menée de manière autonome sur
le projet LFD. Une session possède une spec datée dans [`specs/`](./specs/) : périmètre, ordre,
branches, tests, reviews, déploiements autorisés, mesures, livrables et conditions d'arrêt.

## Règles permanentes

- une branche et une PR par levier ou lot fonctionnel ;
- aucun mélange de changements sans rapport ;
- `staging` uniquement tant qu'une autorisation explicite ne vise pas `main`/prod ;
- aucun merge si les tests, la CI ou les invariants métier ne sont pas verts ;
- l'archive sert de référence de lecture, jamais de source à cherry-pick en bloc ;
- chaque résultat est ajouté aux learnings et au suivi transversal ;
- une mesure de performance n'est valide qu'avec scénario, données, environnement et seuils écrits.

## Session courante

- [2026-07-20 — Chantier A puis B0/B1](./specs/2026-07-20-chantier-a-puis-b0-b1.md)
- [2026-07-20 — Entrées B0/B1 pour Gotenberg privé](./specs/2026-07-20-b0-b1-gotenberg-entrees.md)
- [2026-07-20 — Protocole de validation des générateurs k6](./specs/2026-07-20-protocole-generateur-k6.md)
- [Modèle obligatoire des rapports de load tests](./rapports-load-tests/README.md)
