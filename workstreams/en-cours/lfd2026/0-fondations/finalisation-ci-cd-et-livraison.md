# Finalisation CI/CD et livraison

> Date : 2026-07-13 (déplacé le 13/07 depuis `workspace-rabie/` vers le dossier du chantier 0-fondations)
> Contexte : la partie « tests » du doc [ci-cd-coordination.md](../../../../workspace-rabie/codex-claude/ci-cd-coordination.md) est désormais pilotée par le chantier T (et H pour les sessions). Ce fichier trace l'ordre de travail pour **le reste** : prérequis ops, validation du pipeline, nettoyage, verrouillage.
>
> Logique : **débloquer les prérequis ops d'abord, valider le pipeline ensuite, nettoyer en dernier.**

---

## 1. Prérequis ops (bloquants, rien ne marche sans)

- [x] **Clé SSH dédiée GHA** ✅ 2026-07-13 : ed25519 générée (`~/.ssh/gha_ems_deploy` sur iMac, commentaire `github-actions-deploy-ems`), clé publique ajoutée à `debian@51.75.252.74`, connexion testée OK, clé privée poussée dans le secret `VPS_SSH_KEY` des repos **attendee-ems-back** et **attendee-ems-front** via `gh secret set`.
- [x] **Secrets GitHub Actions** ✅ 2026-07-13 (repos back + front, via `gh secret set`) :
  - `VPS_HOST` = `51.75.252.74` ✅
  - `VPS_USER` = `debian` ✅
  - `VPS_SSH_KEY` = clé privée `gha_ems_deploy` ✅
  - `NOTIFY_WEBHOOK` = optionnel, **pas encore créé** (à faire si on veut les notifs)
  - Vérifié : les workflows `chore/ci-cd` (back : ci/cd-staging/cd-prod ; front) référencent exactement ces noms + `GITHUB_TOKEN` (auto).
- [x] **PAT GHCR côté VPS** ✅ 2026-07-13 : PAT classic `read:packages` (exp. 13/07/2027), `docker login ghcr.io` réussi **en root ET en debian** (credentials dans `/root/.docker/config.json` et `/home/debian/.docker/config.json`). Pull testé : `manifest unknown` = auth OK, l'image `ghcr.io/rabiegha/ems-api` n'existe pas encore (CI jamais lancée — premier run la créera).
- [x] **Droits de `debian` vérifiés** ✅ 2026-07-13 : membre du groupe `docker` (docker **sans sudo** : OK), `sudo` sans mot de passe : OK, `git fetch` : OK, `.env.production` lisible sans sudo : OK. → **Option A déjà en place**, aucun changement de script nécessaire.
  - ⚠️ Rappel : jamais `sudo bash` sur tout le script → on perd la config SSH/Git de `debian` (`Permission denied (publickey)`).

→ ~1h de manip côté Rabie/VPS, tout le reste en dépend.

## 2. Audit nginx / dossier legacy (lecture seule, sans risque — parallélisable avec le 1)

- [x] **Audit fait ✅ 2026-07-13 — verdict : `/opt/ems-attendee/backend/frontend` est un dossier MORT.**
  - Bind réel du conteneur : `docker inspect ems-nginx` → `/opt/ems-attendee/frontend/dist -> /usr/share/nginx/html` (confirmé aussi dans `docker-compose.prod.yml:82`).
  - Preuve par le hash : la prod sert `assets/index-DdGYVq3D.js`, présent **uniquement** dans `frontend/dist/assets/` (1 seul bundle). `backend/frontend/assets/` contient plusieurs vieux bundles accumulés (`index-374pagEj.js`, `index-5KepezVR.js`, …) → copies successives jamais servies.
  - Références restantes : **aucun conteneur ne monte** `backend/frontend`, **aucune conf nginx** ne le cite. Seuls les 4 scripts legacy y copient encore : `deploy-back-logs.sh`, `deploy-front.sh`, `deploy-logs.sh`, `deploy.sh` → étapes mortes.
  - Taille : **912 Mo** à récupérer lors du nettoyage (point 4).
- [x] Constat établi — rien supprimé à ce stade, conformément au plan. Le CD front peut être validé : le swap de `/opt/ems-attendee/frontend/dist` est bien le bon endroit.

### 2bis. Découvertes de l'audit (2026-07-13) — à traiter

- [x] **Vhost staging DISPARU → restauré** ✅ : `nginx/conf.d/staging.attendee.fr.conf` avait été balayé par un changement de branche sur le VPS (il n'existe que dans le commit `3f0a69d`, jamais mergé dans dev/staging/main). staging.attendee.fr répondait **404** depuis. Restauré via `git checkout 3f0a69d -- nginx/conf.d/staging.attendee.fr.conf` + `nginx -s reload` (zéro coupure). Vérifié : staging 200, staging API 200, prod 200, prod API 200.
- [x] **⚠️ Commiter le vhost staging dans les branches** ✅ 2026-07-13 : commit `770695a` poussé sur `dev`, cherry-pick sur `staging` (`8c39218`) et `main` (`1e98065`). Checkout VPS synchronisé (`git pull --ff-only` → `8c39218`, working tree propre, vhost désormais **tracké**). Au passage : le `docker-compose.staging.yml` untracked du VPS a été remplacé par la version de la branche (plus récente, support `EMS_API_IMAGE` pour le CD — backup dans `/tmp/docker-compose.staging.yml.vps-backup`). nginx -t OK, staging 200.
- [x] **CD staging front créé** ✅ 2026-07-13 : `cd-staging.yml` ajouté sur `chore/ci-cd` front (commit `bca6eca`). Auto sur CI verte branche `staging` + dispatch manuel, build GHA avec `VITE_API_BASE_URL=https://staging.attendee.fr/api`, swap atomique de `dist/staging` avec snapshot rollback, **reload nginx à chaud** (zéro coupure prod).
- [x] **🔴 Bug du swap prod corrigé** ✅ 2026-07-13 (même commit `bca6eca`) : `cd-prod.yml` déplace maintenant `dist/staging` dans `dist.new` **avant** le swap, et le sauve avant le `rm -rf` du rollback. Le staging survit aux deploys prod. (Choix : on garde `dist/staging` en sous-dossier plutôt qu'un dossier frère — pas de changement de vhost ni de compose nécessaire.)

## 3. Validation du pipeline (séquentiel, dans cet ordre)

> Découverte 13/07 : `chore/ci-cd` back était **déjà mergé dans `staging`** — la CI tournait déjà sur les push staging.

- [x] **🔧 Fix CI e2e (bloquant, résolu)** ✅ 2026-07-13 : le job `e2e-tests` pendait indéfiniment (3h49 sur le run du 12/07, kill par GHA). Cause : le `.env.test` généré par la CI ne définissait pas `REDIS_HOST`, or le code a `default('redis')` (hostname docker-compose) → sur le runner GHA (service redis sur `localhost:6379`), BullMQ bouclait en `EAI_AGAIN` et jest ne sortait jamais. Fix : `REDIS_HOST=localhost` + `REDIS_PORT=6379` dans le `.env.test` + `timeout-minutes: 15` sur le job. Commits : `f7bb577` (chore/ci-cd) + `c07b38f` (staging).
- [x] Note : `build-image` était déjà vert → **la première image GHCR `ghcr.io/rabiegha/ems-api` existe** (poussée par les runs staging).
- [x] **🔧 Fix CI e2e #2 : seed manquant** ✅ 2026-07-13 : sur base CI vierge, 12/18 tests échouaient en 401 — les specs supposent des users pré-existants (`admin@acme.test`, `admin@acme.com`) **jamais créés par aucun seeder du repo** (le hash de `seed-dev.sql` ne correspond même pas à `password123`). Fix : `test/seed-e2e.ts` (réutilise les seeders dev + rôle `staff` + les 2 users e2e en ADMIN@Acme) + step `Seed E2E` dans `ci.yml`. Commits : `a84f59b` (chore/ci-cd) + `ef9eff3` (staging).
- [x] **🔧 Fix CI e2e #3 : tests obsolètes + throttler** ✅ 2026-07-13 : 3 derniers échecs — (a) le throttler `@Throttle(10/min)` de `/auth/login` déclenchait des 429 (les suites enchaînent >10 logins) → `skipIf: NODE_ENV === 'test'` dans le ThrottlerModule ; (b) assertions cookie obsolètes (`Path=/auth/refresh` vs `Path=/` réel, `Max-Age=0` vs `Expires=1970` réel) → alignées sur le code. Commits : `72b9be1` (chore/ci-cd) + `28eb5d2` (staging). → **CI verte 18/18** ✅
- [x] **⚠️ Workflows portés sur `main`** ✅ 2026-07-13 : découverte — GitHub n'active `workflow_run` ET `workflow_dispatch` que si le workflow existe sur la **branche par défaut**. Le cherry-pick sélectif partait en conflits en cascade (main très en retard) → **merge `staging` → `main`** (commit `4362745`, validé par Rabie). Aucun déploiement auto déclenché (CD prod = dispatch + confirm DEPLOY).

1. [x] **CD staging back** ✅ 2026-07-13 : dispatch manuel → run `29266715214` **success**. Vérifié sur le VPS : `ems-staging-api` tourne sur `ghcr.io/rabiegha/ems-api:28eb5d2…` (healthy), `https://staging.attendee.fr/api/health` → status ok, db ok, redis ok, migrations ok, version = sha déployé. **Chaîne complète CI → GHCR → CD → VPS validée.** (Note : le déclenchement auto `workflow_run` sera actif dès le prochain push staging, maintenant que le workflow est sur main.)
2. [x] **CD prod back ✅ 2026-07-13 ~23h** — validé en 3 runs, avec un incident instructif :
   - **Run 1 (`b50339a`) : ÉCHEC + prod 502 ~7 min.** L'API démarrait parfaitement mais 2 bugs latents : (a) le **healthcheck compose** utilisait `wget -4` (flag inexistant en BusyBox → conteneur `unhealthy` à vie — fix qui existait dans `staging-archive-2026-06-25` mais avait été perdu) ; (b) **nginx ne re-résout pas l'upstream `api`** après recréation du conteneur (nouvelle IP) → 502. Le healthcheck du script (via nginx) échouait donc alors que l'API était saine, et « premier deploy registry » = **pas de SHA de rollback** → exit 1, intervention manuelle. Restauré par `nginx -s reload` (prod re-up immédiatement).
   - **Fixes commités (`4dd9092`)** : reload nginx intégré à `deploy_image()` dans `deploy-from-registry.sh` + healthcheck compose sans `-4`. `deployed_sha` écrit manuellement sur le VPS (rollback désormais possible).
   - **Run 2 (`8963315`) : deploy VPS OK** (reload nginx efficace) mais step « Healthcheck public » KO — URL erronée `api.attendee.fr/health` (il manque le préfixe global `/api`) → 404. Fix : `/api/health` (`aca1b49`).
   - **Run 3 (`aca1b49`) : 100 % VERT** — guard, risky-migration-check, verify-ci, deploy (1m20), notify. Prod : `https://api.attendee.fr/api/health` → status ok, version = sha exact. **La chaîne CD prod back est rodée, rollback armé.**
3. [ ] ⏸ **CD prod front — EN ATTENTE** — le CD prod back est validé, plus de bloqueur. Fenêtre 22h-06h.

### 3bis. Alignement des branches (2026-07-13 soir) ✅

- [x] **Back** : cherry-pick prettier `8b53222` → `staging` (`0aa540f`) puis merge `staging → main` (`b50339a`). `staging` ⊇ `chore/ci-cd` (seul écart résiduel : le vhost nginx staging, présent dans staging/main uniquement — normal).
- [x] **Front** : merge `chore/ci-cd → staging` (`c274cf4`, conflits `eventsApi.ts`/`EventSessionsTab.tsx` résolus : version sessions conservée + prettier réappliqué, tsc/build/eslint verts) puis fast-forward `staging → main`. **La chaîne CI/CD front est désormais sur `main`** (workflows actifs sur la branche par défaut).
- [x] Résultat : **staging = main sur les deux repos** (au commit près), stack staging (`infra/staging-stack`) intégrée partout. Branches `chore/ci-cd` (back+front) et `infra/staging-stack` supprimables.

### 3ter. Rodage CI + CD staging front en conditions réelles (2026-07-13 nuit) ✅

- [x] **Premier vrai run CI front = 2 échecs corrigés** (commit `498dd45`, staging+main) : (a) vitest ramassait les specs **Playwright** de `tests/e2e/` (« test.describe() not expected ») → `exclude` + `passWithNoTests` (constat : **0 test unitaire vitest dans le repo**, les 2 specs sont Playwright — à combler via chantier T) ; (b) directive `eslint-disable react-hooks/exhaustive-deps` devenue inutile dans `AdvancedExportModal.tsx` → supprimée.
- [x] **Chaîne auto front validée de bout en bout** : push staging → CI verte → `workflow_run` → **CD staging front success** (bundle → VPS, swap atomique `dist/staging`, healthcheck public OK). staging.attendee.fr **200**, prod **200** (intacte).
- [x] Confirmé au passage : le **CD staging back auto** s'est aussi déclenché sur le push staging de la soirée — l'API staging tourne sur `0aa540f` (`/api/health` : db/redis/migrations ok). Le `workflow_run` auto est rodé **des deux côtés**.

## 4. Nettoyage (seulement une fois le CD rodé)

- [ ] Renommer le dossier legacy plutôt que supprimer :

  ```bash
  mv /opt/ems-attendee/backend/frontend /opt/ems-attendee/backend/frontend.legacy-2026-07-13
  docker compose -f /opt/ems-attendee/backend/docker-compose.prod.yml restart nginx
  curl -I https://attendee.fr
  ```

  → si le front public reste OK après quelques jours, supprimer le dossier legacy.

- [ ] Corriger les scripts legacy (`deploy-front.sh`, `deploy.sh`, `deploy-logs.sh`, `deploy-back-logs.sh`) : supprimer la copie morte vers `backend/frontend`, mais **les garder comme fallback manuel** pendant l'event (reco Codex, validée).
- [ ] Rangement `scripts/deploy-from-registry.sh` → `scripts/deploy/` : cosmétique. Si fait, mettre à jour `cd-prod.yml` / `cd-staging.yml` **dans le même commit** + vérifier les chemins docs et les commandes VPS.

## 5. Verrouillage final

- [ ] 🔴 **Branch protection sur `main` : BLOQUÉE par le plan GitHub Free** (constat 13/07 : branch protection ET rulesets → HTTP 403 « Upgrade to GitHub Pro » sur repo privé). Options : **GitHub Pro ~4 $/mois sur le compte Rabiegha** (recommandé, débloque tous les repos privés) · org Team (~4 $/user/mois, transfert des repos) · repos publics (non recommandé). Les commandes `gh api` sont prêtes (checks requis identifiés : back `lint-typecheck`/`unit-tests`/`e2e-tests`/`build-image`, front `lint-typecheck`/`unit-tests`/`build`/`e2e-smoke`). `gh` CLI installé + authentifié le 13/07.
- [ ] Chantier **0-MON / monitoring** si pas encore actif.

---

## Résumé de l'ordre

```txt
1 + 2 en parallèle dès maintenant (secrets/clé SSH d'un côté, audit lecture seule de l'autre)
→ 3 séquentiellement (staging back → prod back → prod front)
→ 4 puis 5 une fois le CD rodé
```

## Rappel du chemin principal visé

```txt
Back prod  : GHA cd-prod.yml → SSH VPS → scripts/deploy-from-registry.sh prod <sha>
Front prod : GHA cd-prod.yml front → build dist sur GHA → upload dist.tar.gz → swap /opt/ems-attendee/frontend/dist
Scripts legacy = fallback manuel temporaire uniquement
```
