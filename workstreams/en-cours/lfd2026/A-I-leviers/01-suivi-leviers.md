# 01 — Suivi des leviers (chantier A refonte) — LFD 2026

> **À QUOI SERT CE FICHIER :** c'est le **tableau de bord vivant** de la refonte propre (chantier A).
> Source de vérité de **ce qui est fait / ce qui reste**, levier par levier, branche par branche.
> **À ouvrir au début de chaque nouveau chat** (on fait 1 chat par levier) et sur n'importe quel poste.
>
> - Le **quoi/pourquoi** (cadrage, temps estimés) → [00-plan-action.md](./00-plan-action.md) §0–§2.
> - Le **focus global** (tous chantiers) → [../../../workspace-rabie/NOW.md](../../../workspace-rabie/NOW.md).
> - Les **learnings** produits → [../../../learnings/README.md](../../../learnings/README.md).
>
> **Principe maître :** _ne rien perdre, tout rejouer proprement_. 1 levier = 1 branche = 1 commit
> (diff minimal) = 1 PR = 1 mesure avant/après. Archive de référence : branche `staging-archive-2026-06-25`
>
> - tag `archive/staging-2026-06-25` (back & brain).
>
> **Dernière mise à jour :** 2026-07-10

---

## Conventions à respecter (rappel, pour tout nouveau chat)

- ✅ **Pas de doc infra dans `attendee-ems-back`** → toute doc va dans `attendee-ems-brain`.
- ✅ **Format-on-save OFF** pendant tout le chantier (jamais `npm run format` / Format Document) → diffs chirurgicaux.
- ✅ **1 levier = 1 commit propre** sur sa **branche dédiée**.
- ✅ **Workflow** (décision 2026-07-10) : branche dédiée → test **local** → **MR vers `staging`**
  (branche d'environnement : le serveur staging est construit **depuis `staging`**) → rebuild
  api sur le VPS + campagne **k6** → en fin de chantier, **merge `staging` → `dev`**.
  `dev` = espace des 2 devs, `staging` = espace de l'environnement staging.
  ✅ **Branche `staging` recréée proprement le 2026-07-10** depuis `main` (`c10a4fa`).
  L'ancienne (fourre-tout ressuscité par un `git pull` le 09/07, merge `e2d90ee`) est archivée :
  branche `staging-archive-2026-07-10` + tag `archive/staging-2026-07-10`. Checkouts VPS reset
  (prod → `main`, clone staging → nouvelle `staging`). ⚠️ Reste : le poste de Corentin doit
  reset sa `staging` locale (`git fetch && git checkout staging && git reset --hard origin/staging`)
  et merger `feat/qr-hmac-security` via PR (jamais de push direct).
- ✅ **Confirmer avant toute action distante/destructive** (push, force-push, delete). Commits locaux = OK.
- ✅ **Jamais `--amend`/`rebase` sur un commit déjà poussé d'une branche partagée** (cf. learning git).
- ✅ Nettoyage historique perso = `reset --soft` + `push --force-with-lease` (jamais `--force` seul).

---

## Tableau de bord — leviers

> Ordre du plan ≠ ordre d'exécution : on a fait **L1/L2/L10 en premier** (fondation qui sert à
> **tester** tous les autres sur la stack staging). Le reste suit l'ordre du plan.

| Levier                                                           | Branche                        | Nature                                   | Statut                                    | MR                                                                                    | Mesure k6                                                                            |
| ---------------------------------------------------------------- | ------------------------------ | ---------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **L1/L2/L10** stack staging + pgBouncer + pool                   | `infra/staging-stack`          | infra/config                             | � **Sur `staging` + mesuré** (2026-07-10) | [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) **mergée sur `staging`** | ✅ avant/après fait → [détail ci-dessous](#-l1l2l10--stack-staging--pgbouncer--pool) |
| **L9** transaction allégée (`public.service.ts`)                 | `feat/register-tx-slim`        | **code métier** (chirurgie)              | ⚪ À faire                                | —                                                                                     | —                                                                                    |
| **L9.1** compteur présence session O(1) (candidat, **priorisé**) | `feat/session-present-counter` | code métier (colonne PG `present_count`) | ⚪ À faire                                | —                                                                                     | —                                                                                    |
| **L7** email async BullMQ                                        | `feat/email-async-bullmq`      | nouveaux fichiers                        | ⚪ À faire                                | —                                                                                     | —                                                                                    |
| **L8** worker `PROCESS_ROLE` (Voie A)                            | `feat/process-role-worker`     | code (gating `main.ts`)                  | ⚪ À faire                                | —                                                                                     | —                                                                                    |
| **L3** `directUrl` Prisma                                        | `chore/prisma-directurl`       | config (+5 lignes)                       | ⚪ À faire                                | —                                                                                     | —                                                                                    |

**Légende statut :** ⚪ à faire · 🟡 en cours · 🟢 codé+testé local · 🔵 sur `dev` · 🟣 sur `staging`+mesuré · ✅ clos.

---

## Étapes transverses du chantier A

| #   | Étape                                            | Statut               | Note                                                                                                                                                                                                                 |
| --- | ------------------------------------------------ | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0   | Geler & archiver                                 | ✅ Fait              | tag `archive/staging-2026-06-25` + branche `staging-archive-2026-06-25` (back). Brain : à vérifier.                                                                                                                  |
| 1   | Séparer reformatage / logique                    | ✅ Fait              | format-on-save vérifié OFF (learning créé).                                                                                                                                                                          |
| 2   | Rejouer les leviers (chantier A + L9.1)          | 🟡 En cours          | 1/6 fait — **L1/L2/L10 mergé sur `staging` + mesuré avant/après (2026-07-10)**. Suivants : L9 + L9.1.                                                                                                                |
| 3   | Réconcilier la doc (~25/s CPU vs ~33/s DB)       | ⚪ À faire           | dans `infra-scaling-pca/README.md`.                                                                                                                                                                                  |
| 4   | Post-mortem 502 prod                             | ✅ Fait (2026-07-10) | [bugs/fait/2026-06-25-prod-502-collision-compose.md](../../../bugs/fait/2026-06-25-prod-502-collision-compose.md) + garde-fous suivis dans [garde-fous-deploiement-staging.md](./garde-fous-deploiement-staging.md). |
| 5   | Isoler le compose prod (PR dédiée)               | ⚪ À faire           | `docker-compose.prod.yml`, ne pas merger sans revue.                                                                                                                                                                 |
| 6   | Cadrer l'interdit (L13 cluster, L12 sync_commit) | ⚪ À faire           | note, pas de code.                                                                                                                                                                                                   |

---

## Détail par levier

### � L1/L2/L10 — stack staging + pgBouncer + pool

- **Branche :** `infra/staging-stack` · **Commit :** `fd02111` (unique, propre) · **MR :** [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) **mergée sur `staging`** (2026-07-10, merge `3c418cc`).
- **Fichiers :** `docker-compose.staging.yml`, `.env.staging.example`, `scripts/staging-make-env.sh`.
- **Fait :**
  - Stack iso-prod isolée (réseau `ems-staging-network`) : postgres 16 + pgBouncer (transaction, pool 100) + redis + mailpit + api.
  - **Testée en local** : tous conteneurs healthy, API `/api/health` = **200**.
  - **2 bugs corrigés** dans `staging-make-env.sh` pendant le test : chemin `cd .../..` (le fichier avait été déplacé dans `scripts/`) + `sed` portable BSD/GNU (macOS).
  - Historique **nettoyé** (doublon `--amend`-après-push supprimé via `reset --soft` + `force-with-lease`).
  - Doc de mise en route : **uniquement** sur le brain (`infra/staging/staging-setup.md`), retirée du back.
- **Reste :**
  - [x] Recibler la MR #14 sur la nouvelle `staging` (propre, = `main`) puis la merger → fait le 2026-07-10 (merge `3c418cc`).
  - [x] Déployer sur **staging** (`cd /opt/ems-attendee/staging/backend && git pull` + rebuild api) puis lancer la **baseline k6** → fait le 2026-07-10 (voir mesures ci-dessous).

#### 📊 Mesures k6 avant/après (2026-07-10, `register-baseline.js`, 50 VUs, 3 min, event LFD `published`)

| Métrique (register) | AVANT (code propre, DB **directe** 5432) | APRÈS (pgBouncer 6432 + pool L2/L10) |
| ------------------- | ---------------------------------------- | ------------------------------------ |
| Succès              | 3 595 / 3 595 (**100 %**)                | 3 578 / 3 578 (**100 %**)            |
| p95                 | 162 ms                                   | **149 ms**                           |
| Médiane             | 60 ms                                    | 68 ms                                |
| Moyenne             | 72 ms                                    | 78 ms                                |
| Débit               | ~19,9 inscr./s                           | ~19,8 inscr./s                       |

- **Lecture honnête :** à charge douce (~20 req/s, limitée par le think time du script), **équivalent** —
  pgBouncer ajoute un petit hop (+8 ms médiane) mais lisse la queue (p95 −13 ms). Le vrai bénéfice
  L2/L10 se mesurera **sous stress** (tempête de connexions, `register-stress.js` / `spike-absorption.js`)
  — à rejouer quand les leviers métier (L9…) seront posés.
- **Conditions :** image API rebuiltée `--no-cache` depuis code propre · prod vérifiée **200 avant/après chaque run** ·
  event `completed` → 403 au register (testé), la baseline utilise l'event LFD `published` (token `bAYZDkot7gXfG8FC`).
- **⚠️ 2 pièges découverts pendant la campagne :**
  1. **P1002 advisory lock** : avec `DATABASE_URL` → pgBouncer (pooling transactionnel), `prisma migrate deploy`
     au boot (`RUN_MIGRATIONS=true` dans l'entrypoint) **crash-loop** (advisory lock impossible). Fix temporaire :
     `RUN_MIGRATIONS=false` sur staging. **Fix propre = L3 `directUrl`** — argument massue pour ce levier.
  2. **Cache Docker pollué** : un `up -d --build` après le build de la staging polluée a resservi des layers
     contenant du code d'une autre branche (throttler QR HMAC → 429 sur le run). Toujours vérifier le contenu
     de l'image après changement de branche, ou builder `--no-cache`.
- **Note :** `staging` contient aussi le **chantier D** (sécurité QR, PR #17 mergée via `dev` par Corentin le 10/07) —
  dont un throttler global (Redis) + `@Throttle(limit: 100/min)` sur `POST /register`. **Les prochains runs k6
  seront throttlés** → prévoir de relever la limite en staging ou un bypass dédié aux tests de charge.
- **Learnings liés :** [stack staging](../../../learnings/2026-07-07-stack-staging-concepts-infra.md) · [pgBouncer/pool](../../../learnings/2026-07-07-pgbouncer-et-pool-db.md) · [directUrl](../../../learnings/2026-07-07-directurl-prisma.md) · [git checkout/reset](../../../learnings/2026-07-07-git-checkout-ref-fichier.md).

### ⚪ L9 — transaction d'inscription allégée

- **Branche :** `feat/register-tx-slim` · **Fichier :** `src/.../public.service.ts` (~50 lignes utiles).
- **Nature :** ⚠️ **vraie chirurgie de code** → **test de non-régression obligatoire** (inscriptions identiques, pas de doublon, quota respecté).
- **Source de rejeu :** archive `staging-archive-2026-06-25` (rejouer à la main, pas cherry-pick du fourre-tout).
- **Reste :** tout. À démarrer dans un chat dédié.

### ⚪ L9.1 — compteur de présence session O(1)

- **Branche :** `feat/session-present-counter` · **Fichier :** `src/modules/sessions/sessions.service.ts`.
- **Problème :** `2× COUNT(*)` sur `session_scans` à chaque scan IN → **O(n)** (coût qui grandit avec la table).
- **Levier :** compteur incrémental → **O(1)**. **Colonne PG `present_count` par défaut** (suffit, faible concurrence) ; Redis en **bonus « win no effort »** si affichage live voulu (Redis déjà là pour l'inscription).
- **Nature :** ⚠️ **code métier** → **test de non-régression obligatoire** (pas de survente, occupation correcte après IN / OUT / annulation, idempotence double-scan).
- **Conditionné mesure** à l'origine, **priorisé** pour ce cycle (décision 2026-07-08).
- **Détail conceptuel :** [workstream 02 §2g](../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md).
- **Reste :** tout. À démarrer dans un chat dédié (après/avec L9).

### ⚪ L7 — email async (BullMQ)

- **Branche :** `feat/email-async-bullmq` · **Nature :** nouveaux fichiers (rejeu guidé).
- **But :** sortir l'envoi d'email du chemin critique de l'inscription (file BullMQ).
- **Reste :** tout.
- **Branche :** `feat/email-async-bullmq` · **Nature :** nouveaux fichiers (rejeu guidé).
- **But :** sortir l'envoi d'email du chemin critique de l'inscription (file BullMQ).
- **Reste :** tout.

### ⚪ L8 — worker `PROCESS_ROLE` (Voie A)

- **Branche :** `feat/process-role-worker` · **Nature :** gating dans `main.ts`.
- **But :** pouvoir lancer un process dédié « worker » (consomme la file) distinct de l'API.
- **Reste :** tout.

### ⚪ L3 — `directUrl` Prisma

- **Branche :** `chore/prisma-directurl` · **Nature :** +5 lignes dans `datasource` de `schema.prisma`.
- **But :** URL directe (port 5432) pour les migrations, l'app passant par pgBouncer (6432).
- **⚠️ Urgence montée d'un cran (2026-07-10) :** sans `directUrl`, `prisma migrate deploy` via pgBouncer
  **crash-loop en P1002** (advisory lock) — vécu sur staging pendant la campagne k6. Workaround actuel :
  `RUN_MIGRATIONS=false` dans `.env.staging` (les migrations doivent être lancées à la main en direct).
- **Reste :** tout. Levier le plus simple. Learning déjà écrit.

---

## 🚫 Interdits / en attente (ne pas rejouer)

- **L13 cluster (Voie B)** — **INTERDIT en prod** tant que le workstream `api-scaling-clustering`
  n'est pas livré (Redis-adapter + présence Redis + sticky nginx + tests cross-worker verts).
  Sinon casse l'impression temps réel **silencieusement**.
- **L12 `synchronous_commit`** — abandonné, **aucun gain mesuré**.

---

## Rappel — les 5 moments clés d'un test k6

1. **Baseline** (avant tout levier) — point de départ chiffré, palier de rupture.
2. **Après chaque levier mesurable** — 1 seul changement entre 2 mesures = gain attribuable.
3. **Calibrage** d'un réglage (pool size, workers…) — balayage de valeurs → sommet de la courbe.
4. **Non-régression** (leviers de code) — comportement métier intact.
5. **E2E final** — simulation du pic réel (~12 400 inscriptions) avec marge, avant le jour J.

> k6 se lance **sur staging** (iso-prod), **pas en local** (machine dev fausse les chiffres).
