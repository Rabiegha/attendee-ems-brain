# 04 — État des lieux des branches (back + front)

> **À QUOI SERT CE FICHIER :** photo de l'état des branches git des 2 repos applicatifs,
> avec les actions faites / à faire. À mettre à jour à chaque opération de branches
> (merge structurant, suppression, création de branche de chantier).
>
> **Circuit acté** ([0-CI-plan](./0-fondations/0-CI-plan.md)) : `feature → staging → main` — `dev` sorti du circuit.
> **Règle branche longue durée** : merger `staging` dedans régulièrement (jamais `main` directement)
> pour que le merge final vers `staging` soit trivial.

## Status

- **Type :** SNAPSHOT (photo datée — à confirmer contre `git fetch` avant toute décision)
- **Dernière vérification :** 2026-07-14 (nuit du 13 au 14, après validation CD prod + nettoyage)

---

## Synthèse

- ✅ **`staging` = `main` sur les deux repos** (contenu strictement identique, vérifié par diff d'arbres —
  `main` back porte juste les commits de merge en plus).
- ✅ **`chore/ci-cd` supprimée** (back + front) le 14/07 — tous les patches vérifiés présents dans staging
  (`git cherry` : 0 patch manquant).
- ✅ **`chore/gotenberg-cloudrun-poc` réalignée** = staging (`72d97f6`, fast-forward) — prête pour le code B0.
- ⚠️ **Branche H back de Corentin : JAMAIS poussée sur GitHub** (vérifié via le log d'activité du repo :
  aucune création/suppression de branche session). Elle vit en local sur sa machine. **À faire pousser.**
- 🔴 **Front H en prod appelle un endpoint back inexistant** : `GET/POST /events/:id/sessions/:id/public-token`
  n'existe sur aucune branche back → la feature « lien public de session » est cassée en prod (404 au clic,
  confinée à l'onglet Sessions). Se résout quand Corentin pousse + merge son back H.

---

## Back — `attendee-ems-back`

| Branche | État (14/07) | Action |
| --- | --- | --- |
| `main` | = staging (contenu identique) · prod déployée dessus (`aca1b49`+) | — |
| `staging` | Base d'intégration · CD staging auto à chaque push | — |
| `chore/ci-cd` | ~~4 patches, tous équivalents dans staging~~ | 🗑️ **Supprimée 14/07** |
| `chore/gotenberg-cloudrun-poc` | = staging (`72d97f6`) — aucun commit propre (le POC vit dans le brain) | Prête pour B0 |
| `infra/staging-stack` | Entièrement mergée (PR #14 → partout) | 🗑️ Supprimable |
| `feature/db-backup` | Mergée (PR #15) | 🗑️ Supprimable |
| `feat/qr-hmac-security` | Mergée (PR #16/#17) | 🗑️ Supprimable |
| `dev` | Vestige — sorti du circuit par l'arbitrage 0-CI | À supprimer après confirmation Corentin |
| `staging-archive-2026-06-25` / `-07-10` | Archives volontaires (force-push du 10/07) | **Garder** |
| `stable-branch` / `gcp-migration-stable` | Snapshots anciens (09/07) | À trier post-event |
| `context-organization` | Snapshot doc contexte | À trier post-event |

## Front — `attendee-ems-front`

| Branche | État (14/07) | Action |
| --- | --- | --- |
| `main` | = staging (strictement identique) · prod déployée dessus | — |
| `staging` | Base d'intégration · CD staging auto (build `VITE_API_BASE_URL` staging) | — |
| `chore/ci-cd` | ~~Entièrement mergée~~ | 🗑️ **Supprimée 14/07** |
| `dev` | 14 commits en retard sur main, 0 d'avance — vestige | Supprimable |
| `gcp-migration-stable` | 12 en retard, 0 d'avance — snapshot | Supprimable |

> Note front : le travail H de Corentin (2 commits `feat(sessions)` du 13/07) a été **commité
> directement sur `staging`** (pas de branche) et mergé dans `main` le 13/07 au soir → en prod.

---

## Historique des opérations

| Date | Opération |
| --- | --- |
| 2026-07-13 | Back : cherry-pick prettier `8b53222` → staging (`0aa540f`) · merge staging → main (`b50339a`) |
| 2026-07-13 | Front : merge `chore/ci-cd` → staging (`c274cf4`, 2 conflits sessions résolus) · ff staging → main |
| 2026-07-13 | Fixes CI/CD mergés au fil de l'eau (vitest exclude, permissions actions:read, healthchecks, nginx reload) — staging et main maintenus alignés |
| 2026-07-13 | Rangement `scripts/` + `test/` back (`72d97f6`) — validé par CD staging auto |
| 2026-07-14 | 🗑️ Suppression `chore/ci-cd` (back + front) · réalignement gotenberg = staging |
