# 01 — Suivi des leviers (chantier A refonte) — LFD 2026

> **À QUOI SERT CE FICHIER :** c'est le **tableau de bord vivant** de la refonte propre (chantier A).
> Source de vérité de **ce qui est fait / ce qui reste**, levier par levier, branche par branche.
> **À ouvrir au début de chaque nouveau chat** (on fait 1 chat par levier) et sur n'importe quel poste.
>
> - Le **quoi/pourquoi** (cadrage, temps estimés) → [00-plan-action.md](./00-plan-action.md) §0–§2.
> - Le **focus global** (tous chantiers) → [../../../workspace-rabie/NOW.md](../../../workspace-rabie/NOW.md).
> - Les **learnings** produits → [../../../learnings/README.md](../../../learnings/README.md).
>
> **Principe maître :** *ne rien perdre, tout rejouer proprement*. 1 levier = 1 branche = 1 commit
> (diff minimal) = 1 PR = 1 mesure avant/après. Archive de référence : branche `staging-archive-2026-06-25`
> + tag `archive/staging-2026-06-25` (back & brain).
>
> **Dernière mise à jour :** 2026-07-08

---

## Conventions à respecter (rappel, pour tout nouveau chat)

- ✅ **Pas de doc infra dans `attendee-ems-back`** → toute doc va dans `attendee-ems-brain`.
- ✅ **Format-on-save OFF** pendant tout le chantier (jamais `npm run format` / Format Document) → diffs chirurgicaux.
- ✅ **1 levier = 1 commit propre** sur sa **branche dédiée**.
- ✅ **Workflow** : branche dédiée → test **local** (dev) → **MR vers `dev`** → (campagne) **staging** + k6.
- ✅ **Confirmer avant toute action distante/destructive** (push, force-push, delete). Commits locaux = OK.
- ✅ **Jamais `--amend`/`rebase` sur un commit déjà poussé d'une branche partagée** (cf. learning git).
- ✅ Nettoyage historique perso = `reset --soft` + `push --force-with-lease` (jamais `--force` seul).

---

## Tableau de bord — leviers

> Ordre du plan ≠ ordre d'exécution : on a fait **L1/L2/L10 en premier** (fondation qui sert à
> **tester** tous les autres sur la stack staging). Le reste suit l'ordre du plan.

| Levier | Branche | Nature | Statut | MR | Mesure k6 |
|---|---|---|---|---|---|
| **L1/L2/L10** stack staging + pgBouncer + pool | `infra/staging-stack` | infra/config | 🟢 **Codé + testé local** | [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) → `dev` (ouverte) | ⏳ pas encore (baseline à faire sur staging) |
| **L9** transaction allégée (`public.service.ts`) | `feat/register-tx-slim` | **code métier** (chirurgie) | ⚪ À faire | — | — |
| **L7** email async BullMQ | `feat/email-async-bullmq` | nouveaux fichiers | ⚪ À faire | — | — |
| **L8** worker `PROCESS_ROLE` (Voie A) | `feat/process-role-worker` | code (gating `main.ts`) | ⚪ À faire | — | — |
| **L3** `directUrl` Prisma | `chore/prisma-directurl` | config (+5 lignes) | ⚪ À faire | — | — |

**Légende statut :** ⚪ à faire · 🟡 en cours · 🟢 codé+testé local · 🔵 sur `dev` · 🟣 sur `staging`+mesuré · ✅ clos.

---

## Étapes transverses du chantier A

| # | Étape | Statut | Note |
|---|---|---|---|
| 0 | Geler & archiver | ✅ Fait | tag `archive/staging-2026-06-25` + branche `staging-archive-2026-06-25` (back). Brain : à vérifier. |
| 1 | Séparer reformatage / logique | ✅ Fait | format-on-save vérifié OFF (learning créé). |
| 2 | Rejouer les 5 leviers | 🟡 En cours | 1/5 fait (L1/L2/L10). |
| 3 | Réconcilier la doc (~25/s CPU vs ~33/s DB) | ⚪ À faire | dans `infra-scaling-pca/README.md`. |
| 4 | Post-mortem 502 prod | ⚪ À faire | `bugs/2026-06-25-prod-502-collision-compose.md`. |
| 5 | Isoler le compose prod (PR dédiée) | ⚪ À faire | `docker-compose.prod.yml`, ne pas merger sans revue. |
| 6 | Cadrer l'interdit (L13 cluster, L12 sync_commit) | ⚪ À faire | note, pas de code. |

---

## Détail par levier

### 🟢 L1/L2/L10 — stack staging + pgBouncer + pool
- **Branche :** `infra/staging-stack` · **Commit :** `fd02111` (unique, propre) · **MR :** [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) → `dev` (ouverte).
- **Fichiers :** `docker-compose.staging.yml`, `.env.staging.example`, `scripts/staging-make-env.sh`.
- **Fait :**
  - Stack iso-prod isolée (réseau `ems-staging-network`) : postgres 16 + pgBouncer (transaction, pool 100) + redis + mailpit + api.
  - **Testée en local** : tous conteneurs healthy, API `/api/health` = **200**.
  - **2 bugs corrigés** dans `staging-make-env.sh` pendant le test : chemin `cd .../..` (le fichier avait été déplacé dans `scripts/`) + `sed` portable BSD/GNU (macOS).
  - Historique **nettoyé** (doublon `--amend`-après-push supprimé via `reset --soft` + `force-with-lease`).
  - Doc de mise en route : **uniquement** sur le brain (`infra/staging/staging-setup.md`), retirée du back.
- **Reste :**
  - [ ] Merger la MR #14 sur `dev`.
  - [ ] Déployer sur **staging** puis lancer la **baseline k6** (1er point de mesure du chantier).
- **Learnings liés :** [stack staging](../../../learnings/2026-07-07-stack-staging-concepts-infra.md) · [pgBouncer/pool](../../../learnings/2026-07-07-pgbouncer-et-pool-db.md) · [directUrl](../../../learnings/2026-07-07-directurl-prisma.md) · [git checkout/reset](../../../learnings/2026-07-07-git-checkout-ref-fichier.md).

### ⚪ L9 — transaction d'inscription allégée
- **Branche :** `feat/register-tx-slim` · **Fichier :** `src/.../public.service.ts` (~50 lignes utiles).
- **Nature :** ⚠️ **vraie chirurgie de code** → **test de non-régression obligatoire** (inscriptions identiques, pas de doublon, quota respecté).
- **Source de rejeu :** archive `staging-archive-2026-06-25` (rejouer à la main, pas cherry-pick du fourre-tout).
- **Reste :** tout. À démarrer dans un chat dédié.

### ⚪ L7 — email async (BullMQ)
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
