# CI/CD — optimisations POST-EVENT (gestion des tests à l'échelle)

> **Statut : 📦 post-event — rien à faire avant LFD.**
> Réponse à la question « est-ce que ça devient lourd si on a beaucoup de tests en CI ? »
> (13/07). Décision actée pour l'event : **on lance TOUT à chaque PR** (voir
> [chantier T §4bis](../T-tests-event/README.md)). Ce fichier garde les pistes d'optimisation
> pour le jour où la CI dépassera ~10 min.

---

## Comment c'est géré en pratique (rappel)

1. **Pyramide de tests** — la vraie protection : beaucoup d'unitaires (millisecondes), moins
   d'intégration/E2E API (secondes), très peu d'E2E navigateur (les seuls vraiment lents).
   C'est la structure déjà proposée dans le chantier T.
2. **Jobs séparés en parallèle** : `lint` + `unit` + `e2e` tournent en parallèle dans
   GitHub Actions → le temps total = le job le plus lent, pas la somme.
3. **Étagement par branche** : unitaires sur toutes les PR · E2E API sur PR vers
   `staging`/`main` · E2E navigateur (Playwright) au merge ou en nightly. C'est le compromis
   classique.
4. **Sélection automatique des tests affectés** (Nx, Turborepo, Bazel) : ça existe, mais c'est
   de l'outillage monorepo pour des suites de 30+ min. **Overkill pour nous.**

## À notre échelle : aucun souci

Même avec P0+P1 (chantier T) + les tests sessions du chantier H, on sera à **~2-5 min de CI**.

**Règle pratique : on n'optimise que quand la CI dépasse ~10 min.**
Donc : tout lancer à chaque PR, et si un jour Playwright devient lent, on le passe en
« au merge seulement » (point 3 ci-dessus, première marche de l'étagement).

## Déclencheurs pour rouvrir ce fichier

- [ ] CI > ~10 min sur une PR moyenne → appliquer le point 3 (étagement par branche).
- [ ] Suite Playwright > ~5 min → passer Playwright au merge/nightly.
- [ ] Passage en monorepo → seulement là, évaluer le point 4 (tests affectés).
