# Quand fait-on une réconciliation `staging` ↔ `main` ?

> Doc de référence. Explique **le concept** (dans quels cas on ouvre une branche de
> réconciliation dédiée plutôt qu'un merge direct). Pour la réconciliation concrète
> en cours, voir [staging-main-2026-07-20.md](./staging-main-2026-07-20.md).

## Le contexte : deux branches qui vivent en parallèle

- `staging` reçoit en continu des chantiers (features, fixes, refactors, tooling de
  test de charge…) testés sur l'environnement staging avant promotion.
- `main` reçoit historiquement ses propres correctifs directs (CI/CD, hotfix prod,
  backup…) en parallèle, sans toujours repasser par `staging`.
- Résultat : les deux branches **divergent** — chacune a des commits que l'autre n'a pas.

Une divergence n'est **pas un problème en soi**. Elle ne devient un sujet que quand on
veut **remettre `main` à niveau** avec ce qui a été validé sur `staging` (ou l'inverse).

## Dans quels cas on ouvre une branche de réconciliation dédiée

On ne fait **pas** un `git merge staging → main` (ou une PR GitHub directe) à l'aveugle.
On passe par une branche dédiée (ex. `chore/reconcile-staging-main`) dès qu'**au moins
une** de ces conditions est vraie :

1. **`main` déclenche une chaîne de déploiement automatique (CD prod)**
   Un merge direct dans l'UI GitHub ne permet pas de tester le résultat de la fusion
   avant qu'il parte en prod. Une branche à part permet de builder, faire tourner la
   CI et valider en local/staging **avant** de proposer le merge final vers `main`.

2. **Un `git merge --no-commit --no-ff` (simulation) révèle des conflits de contenu**
   Si la fusion automatique échoue sur au moins un fichier, il faut un espace de
   travail dédié pour résoudre chaque conflit avec le contexte métier nécessaire —
   pas la petite fenêtre de résolution de conflits de l'UI GitHub.

3. **Les conflits touchent de la logique métier ("vraie chirurgie de code")**
   Exemple : deux implémentations différentes d'une même transaction (ancienne
   logique vs nouvelle logique transactionnelle). Ça nécessite des **tests de
   non-régression** ciblés, pas juste "accepter une version". Ce n'est plus une
   opération mécanique.

4. **On veut isoler l'historique de la réconciliation**
   Une branche dédiée + une seule PR de réconciliation gardent l'historique de
   `main` lisible : un seul commit/PR documenté "réconciliation staging→main",
   au lieu de mélanger la résolution de conflits avec d'autres changements en cours.

5. **On veut pouvoir abandonner/recommencer sans impact**
   Si la résolution s'avère mauvaise après coup (tests rouges, comportement
   inattendu), on supprime la branche de réconciliation et on repart de zéro — `staging`
   et `main` ne sont jamais touchées entre-temps.

## Dans quels cas un merge direct suffit (pas besoin de branche dédiée)

- La simulation de merge ne remonte **aucun conflit** (fusion automatique propre).
- Les fichiers touchés sont **additifs et autonomes** (config, doc, script isolé) —
  pas de logique métier partagée modifiée des deux côtés.
- Le repo/l'environnement cible n'a **pas** de déploiement automatique déclenché par
  le merge (donc pas de risque immédiat en cas d'oubli de test).

## Méthode pour vérifier si on est dans le cas "réconciliation nécessaire"

```bash
# 1. Diff des deux branches
git fetch origin --prune
git log origin/main..origin/staging --oneline   # commits staging absents de main
git log origin/staging..origin/main --oneline   # commits main absents de staging

# 2. Simulation réelle du merge (aucun push, juste un worktree jetable)
git worktree add -f /tmp/merge-test origin/main --detach
cd /tmp/merge-test
git merge origin/staging --no-commit --no-ff
# → si "CONFLIT (contenu)" apparaît sur un ou plusieurs fichiers => branche dédiée
git merge --abort
cd - && git worktree remove /tmp/merge-test --force
```

Si l'étape 2 ne montre aucun conflit ET que le repo cible n'a pas de CD automatique
→ un merge direct (PR classique) suffit. Sinon → branche de réconciliation dédiée.

## Précédent sur ce repo (`attendee-ems-back`)

Ce pattern a déjà été appliqué une fois : une PR globale `staging → main` (**#34**,
17/07/2026) avait révélé une divergence de contenu. Elle a été **fermée** et
remplacée par une branche dédiée **`chore/reconcile-staging-main`** (PR **#35**),
où les conflits ont été résolus proprement, testés, CI verte — mais **jamais
mergée**, en attente d'une décision explicite (le merge vers `main` peut déclencher
la chaîne CD prod).
