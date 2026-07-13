# 2026-07-13 — Où commiter quoi : branche dédiée vs vraies branches, et le worktree

> Contexte : finalisation CI/CD LFD2026. Deux types de changements sont arrivés en même temps
> et ne devaient PAS aller au même endroit. Au passage, un incident réel : le vhost staging
> avait disparu du VPS à cause d'un simple changement de branche.

## La règle

| Type de changement | Destination | Pourquoi |
| --- | --- | --- |
| Code du chantier (workflows CI/CD, scripts deploy) | **branche dédiée** (`chore/ci-cd`) | Sera revu/mergé en PR comme le reste du chantier |
| Conf d'infra dont le serveur dépend (vhost nginx, compose) | **toutes les vraies branches** (`dev` → `staging` → `main`) | Le VPS fait des `git checkout` de branche : si le fichier n'est tracké que dans une branche/commit orphelin, il **disparaît du disque au prochain checkout** |

## L'incident qui illustre la règle

- Le vhost `nginx/conf.d/staging.attendee.fr.conf` n'existait que dans un commit orphelin
  (`3f0a69d`), jamais mergé dans dev/staging/main.
- Un changement de branche sur le VPS (~10/07) l'a **balayé du disque** → staging.attendee.fr
  en 404 pendant ~3 jours, silencieusement (nginx gardait la conf en mémoire jusqu'au reload
  suivant… qui n'a pas eu lieu, c'est le checkout qui a tout cassé).
- Fix : commit sur `dev` + **cherry-pick** (pas merge — le commit d'origine était un fourre-tout)
  sur `staging` et `main`, puis synchro du checkout VPS.

## Le pattern worktree — travailler sur une branche sans toucher son checkout

Pour commiter sur `chore/ci-cd` alors que la copie locale est sur une branche de travail
(`gcp-migration-stable`) avec potentiellement du travail en cours :

```bash
git worktree add /tmp/front-cicd chore/ci-cd   # checkout séparé dans /tmp
# ... éditer, commit, push depuis /tmp/front-cicd ...
git worktree remove /tmp/front-cicd            # nettoyage
```

Avantages : zéro `git stash`, zéro changement de branche dans le dossier principal, aucun
risque pour le travail en cours. À préférer systématiquement au `checkout` + `stash` quand on
doit faire un commit ponctuel sur une autre branche.

## Leçons annexes du même jour

- `git checkout <sha> -- <fichier>` récupère **un seul fichier** d'un commit fourre-tout — mais
  attention, il **stage** le fichier (le laisser untracked demande un `git reset` derrière).
- Un `git pull` peut échouer silencieusement sur `error: untracked working tree files would be
  overwritten` — quand on filtre la sortie avec `tail -1`, on rate l'erreur. Toujours vérifier
  `git log --oneline -1` après un pull scripté.
- `docker exec ems-nginx nginx -s reload` = zéro coupure, à préférer au
  `docker compose restart nginx` quand on ne change que la conf/le contenu servi.

## Leçons GitHub Actions (validation pipeline du 13/07 au soir)

- **`workflow_run` ET `workflow_dispatch` exigent que le workflow existe sur la
  branche par défaut** (`main`). Un workflow présent uniquement sur `staging` ou une
  branche de feature est invisible : pas de déclenchement auto, et même
  `gh workflow run --ref staging` répond `404 not found on the default branch`.
  → Toute mise en place de CD implique un passage par `main` tôt ou tard.
- **Cherry-pick sélectif vers une branche très en retard = conflits en cascade.**
  Quand `main` a 15 commits de retard sur `staging` et que les commits à porter
  touchent du code app (pas juste de la conf), le merge complet `staging → main`
  est plus sûr qu'un cherry-pick partiel qui risque de laisser `main` non compilable.
  Merger `staging → main` ne déploie rien si le CD prod est dispatch-only.
- **Des tests e2e qui n'ont jamais tourné sur base vierge mentent** : les specs du
  back supposaient des users seedés à la main (mauvais hash dans `seed-dev.sql`,
  user inexistant, rôle `staff` absent des seeders) et des comportements dépassés
  (`Path=/auth/refresh`, `Max-Age=0`). Première vraie exécution CI = 12/18 rouges.
  → Un seed e2e dédié et versionné (`test/seed-e2e.ts`) + assertions alignées sur le
  code réel. Règle : les tests doivent porter leurs propres données.
- **Rate-limiting vs e2e** : un `@Throttle(10/min)` sur `/auth/login` casse des suites
  qui enchaînent les logins. `skipIf: NODE_ENV === 'test'` au niveau du ThrottlerModule
  est le pattern propre (le comportement prod reste testé... en prod/staging).
