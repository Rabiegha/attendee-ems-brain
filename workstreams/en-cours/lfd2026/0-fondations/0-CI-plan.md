# Plan CI/CD (0-CI) — staging + prod, prod en priorité

> **Statut :** ACTIF — remplace les hypothèses obsolètes du plan initial.
> **Date :** 2026-07-11 · **Chantier :** 0-CI ([00-plan-action §P0](../00-plan-action.md))
> **Périmètre :** `attendee-ems-back` + `attendee-ems-front`. Hors scope : mobile (EAS), print-client (Electron).
>
> **Référence détaillée (cible complète) :** [CI_CD_PLAN.md](./CI_CD_PLAN.md) — squelette d'origine, à
> lire pour le détail des jobs/tests. **Ce document-ci arbitre et réconcilie** avec le réel (staging, garde-fous).
>
> **Compagnons :** [garde-fous-deploiement-staging.md](../A-I-leviers/garde-fous-deploiement-staging.md) ·
> [post-mortem 502](../../../bugs/fait/2026-06-25-prod-502-collision-compose.md) ·
> [DEBTS.md](../../../debts/DEBTS.md) (T-10 + sécu SSH).

---

## 0. Principe & arbitrages actés (11/07)

- **Le CI/CD couvre 2 environnements : staging ET prod. La prod est la priorité n°1** (live MEAE).
  Les mécanismes de sécurité (healthcheck + auto-rollback + garde migration) servent précisément à
  rendre le **déploiement prod sûr**, pas à l'éviter.
- **Déclencheur prod : `workflow_dispatch`** — on garde la main sur le _quand_, la machine garantit le _comment_.
- **Garde horaire prod (22h-06h Europe/Paris) conservée**, avec **bypass `hotfix=true`** (jamais coincé en journée).
- **Build dans la CI** (image Docker taguée par SHA → registre GHCR) puis **le VPS `pull`** — plus jamais
  de build sur le VPS (supprime le risque « build cassé en prod »).
- **Pas de freeze dur pendant l'event** : discipline (limiter les déploiements) **mais liberté totale**
  conservée via le chemin `hotfix`. On ne se bloque jamais.

---

## 1. Modèle branche → environnement

```
feature/* ──PR + CI──► staging ──deploy──► ENV STAGING (docker-compose.staging.yml)
                          │
                          └──PR + CI + review──► main ──deploy blindé (dispatch)──► ENV PROD
```

| Branche           | Rôle                    | Déploiement                                       |
| ----------------- | ----------------------- | ------------------------------------------------- |
| `feature/*`       | travail quotidien       | — (CI seulement sur PR)                           |
| `staging`         | branche d'environnement | **CD staging** (auto au merge, ou dispatch léger) |
| `main` (protégée) | production              | **CD prod** (`workflow_dispatch` + gardes)        |

> ✅ **Décidé (11/07) : `feature → staging → main`.** La branche `dev` est **sortie du circuit**
> (ni environnement, ni étape) — on l'ignore complètement.

---

## 2. Protection des branches (gratuit, sans org)

**Aucune organisation requise** (branch protection gratuite sur repo privé perso depuis 2023).

| Règle                            | `staging`    | `main`               |
| -------------------------------- | ------------ | -------------------- |
| Require PR before merge          | ✅           | ✅                   |
| Require status checks (CI verte) | ✅           | ✅                   |
| Require approvals (1)            | ⚪ optionnel | ✅ **1 approbation** |
| Block force-push                 | ✅           | ✅                   |
| Block deletion                   | ✅           | ✅                   |
| Linear history                   | ⚪           | ✅ (recommandé)      |

> ✅ **Décidé : 1 approbation requise sur `main`.** Rabie **et** Corentin sont collaborateurs des 2 repos
> et **s'approuvent mutuellement** les PR (GitHub interdit d'approuver sa propre PR).
> ✅ Couvre le **garde-fou #6** (protéger `staging`) qui aurait évité l'incident du 09/07.

---

## 3. CI — sur chaque PR (back + front)

Détail des jobs → [CI_CD_PLAN.md §2](./CI_CD_PLAN.md). Résumé :

| Job              | Back                                       | Front                             |
| ---------------- | ------------------------------------------ | --------------------------------- |
| lint + typecheck | `npm run lint` + `tsc --noEmit`            | `lint` + `typecheck`              |
| tests unitaires  | `npm test`                                 | `vitest --run`                    |
| **build image**  | `docker build` → **push GHCR** (tag = SHA) | build + (image nginx ou artefact) |
| E2E              | Postgres+Redis GHA + `test:e2e`            | Playwright chromium (smoke)       |

→ **CI verte = image taguée disponible dans GHCR**, prête à être déployée (voir §4).

---

## 4. Build & artefacts : CI → GHCR → VPS pull

- La CI **construit l'image Docker** et la **pousse sur GHCR** (`ghcr.io/rabiegha/ems-api:<sha>`).
- Le déploiement (staging/prod) = le VPS **`docker pull`** l'image taguée puis `up -d` — **zéro build sur le VPS**.
- Rollback = re-`pull` du **SHA précédent** (image déjà présente / re-taguée `:rollback`).
- Prérequis : `GHCR_TOKEN` (PAT `write:packages`) en secret ; login GHCR sur le VPS.

---

## 5. CD staging

- **Déclenchement :** au merge sur `staging` (ou `workflow_dispatch` léger).
- **Étapes :** VPS `pull` image `:sha` staging → `staging-up.sh` (garde-fou #3) → **healthcheck** staging.
- Rôle : **roder le pipeline** et valider une release avant prod. Risque faible.

---

## 6. CD prod (priorité n°1)

- **Déclenchement :** `workflow_dispatch` sur `main`, `confirm=DEPLOY` obligatoire.
- **Inputs :** `confirm` (=`DEPLOY`), `hotfix` (bypass garde horaire), `migration_risky` (confirmer migration destructive).
- **Séquence :**
  ```
  guard (confirm + fenêtre 22h-06h | hotfix) → verify-ci (CI verte sur HEAD main)
    → risky-migration-check (grep DROP/ALTER/TRUNCATE) → deploy (pull image :sha, tag :rollback)
    → healthcheck (/health : db/redis/migrations) → [échec] rollback auto → notify
  ```
- **Garde horaire :** rejette hors 22h-06h **sauf `hotfix=true`** (+ notification d'alerte).
- **Auto-rollback :** si healthcheck KO → re-`pull` `:rollback` + restart → notify.
- **Notifications :** début / succès / rollback / migration risquée (webhook Slack ou email).

---

## 7. Prérequis (à faire avant le CD)

| #   | Prérequis                                                                                        | Réf                                   |
| --- | ------------------------------------------------------------------------------------------------ | ------------------------------------- |
| 1   | **Enrichir `/health`** → `{status, db, redis, migrations, version}` + HTTP 503 si KO             | existe déjà (partiel)                 |
| 2   | **Corriger `deploy-back.sh`** : le `sudo bash` casse la config SSH → `Permission denied`         | [DEBTS T-10](../../../debts/DEBTS.md) |
| 3   | Scripts deploy : sauvegarde SHA (`.last_good_sha`), `pull` image, `prisma migrate status` strict | [CI_CD_PLAN §3.5](./CI_CD_PLAN.md)    |
| 4   | **Secrets** : `VPS_SSH_KEY`, `VPS_HOST/USER`, `GHCR_TOKEN`, `NOTIFY_WEBHOOK` en GitHub Secrets   | —                                     |
| 5   | 🔴 **Roter le mot de passe SSH `Choyou2025@`** (tapé en clair) + clé PrintNode                   | rappel sécu BACKLOG/DEBTS             |

---

## 8. Ordre de livraison (prod visée vite, sans casser l'event)

| #   | Étape                                                                  | Env      | Risque   |
| --- | ---------------------------------------------------------------------- | -------- | -------- |
| 1   | **CI** bloquante sur PR (back + front) + push image GHCR               | —        | nul      |
| 2   | **Branch protection** `staging` + `main` (garde-fou #6)                | —        | nul      |
| 3   | **Prérequis** (§7 : `/health`, fix scripts, secrets, rotation)         | —        | faible   |
| 4   | **CD staging** (pull image + healthcheck) — rode le pipeline           | staging  | faible   |
| 5   | **CD prod** (`workflow_dispatch` + rollback + garde migration + notif) | **prod** | maîtrisé |

---

## 9. Event LFD — discipline, pas de blocage

- **Pas de freeze dur.** On **limite** les déploiements prod pendant l'event, mais le chemin
  **`hotfix=true`** reste **toujours** disponible (liberté totale à toute heure).
- Introduire le **CD prod avant l'event** (rodé sur staging), pas pendant.
- Toute action garde le filet : healthcheck + auto-rollback + notification.

---

## Références

- [CI_CD_PLAN.md](./CI_CD_PLAN.md) — squelette détaillé (jobs, tests E2E, fichiers à créer, plan de vérif).
- [garde-fous-deploiement-staging.md](../A-I-leviers/garde-fous-deploiement-staging.md) — 6 garde-fous anti-collision.
- [post-mortem 502 (25/06)](../../../bugs/fait/2026-06-25-prod-502-collision-compose.md).
- [DEBTS.md](../../../debts/DEBTS.md) — T-10 (deploy script) + sécurité SSH.
- [BACKLOG-TECH.md](../../../backlog/a-faire/BACKLOG-TECH.md) — stack staging, `/health`, `docker-compose.staging.yml`.
