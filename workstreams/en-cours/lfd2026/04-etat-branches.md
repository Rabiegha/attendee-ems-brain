# 04 — État des lieux des branches (back + front + mobile)

> **À QUOI SERT CE FICHIER :** photo de l'état des branches git des repos applicatifs,
> avec les actions faites / à faire. À mettre à jour à chaque opération de branches
> (merge structurant, suppression, création de branche de chantier).
>
> **Circuit acté** ([0-CI-plan](./0-fondations/0-CI-plan.md)) : `feature → staging → main` — `dev` sorti du circuit.
> **Règle branche longue durée** : merger `staging` dedans régulièrement (jamais `main` directement)
> pour que le merge final vers `staging` soit trivial.

## Status

- **Type :** SNAPSHOT (photo datée — à confirmer contre `git fetch` avant toute décision)
- **Dernière vérification :** 2026-07-16 soir (après merge `chore/monitoring` → `staging`, back `63c2bd6`)

---

## Synthèse

- ⚠️ **Back : `staging` est en avance sur `main`** depuis le merge 0-MON (`63c2bd6`) ; à promouvoir en prod
  uniquement après validation VPS/alertes.
- ✅ **Front : `staging` = `main`** (contenu strictement identique à la dernière photo connue).
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

| Branche                                  | État (16/07)                                                           | Action                                  |
| ---------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------- |
| `main`                                   | En retard sur `staging` du merge 0-MON (`63c2bd6`)                     | Garder stable jusqu'aux vérifs VPS      |
| `staging`                                | Base d'intégration · contient 0-MON monitoring ops (`63c2bd6`)         | Déployer/vérifier VPS avant promotion   |
| `chore/ci-cd`                            | ~~4 patches, tous équivalents dans staging~~                           | 🗑️ **Supprimée 14/07**                  |
| `chore/gotenberg-cloudrun-poc`           | = staging (`72d97f6`) — aucun commit propre (le POC vit dans le brain) | Prête pour B0                           |
| `chore/monitoring`                       | Mergée dans `staging` (`63c2bd6`) · diff contenu nul vs `staging`      | ✅ Livrée staging                        |
| `feat/internal-mailing-wave1`            | Diff contenu nul vs `staging` (le warm-up est déjà dans staging via PR #26) | ✅ Rien à merger                         |
| `feat/c2-mailgun-integration`            | Mergée via PR #20 puis supprimée (locale + remote)                     | ✅ Clos                                 |
| `infra/staging-stack`                    | Entièrement mergée (PR #14 → partout)                                  | 🗑️ Supprimable                          |
| `feature/db-backup`                      | Mergée (PR #15)                                                        | 🗑️ Supprimable                          |
| `feat/qr-hmac-security`                  | Mergée (PR #16/#17)                                                    | 🗑️ Supprimable                          |
| `dev`                                    | Vestige — sorti du circuit par l'arbitrage 0-CI                        | À supprimer après confirmation Corentin |
| `staging-archive-2026-06-25` / `-07-10`  | Archives volontaires (force-push du 10/07)                             | **Garder**                              |
| `stable-branch` / `gcp-migration-stable` | Snapshots anciens (09/07)                                              | À trier post-event                      |
| `context-organization`                   | Snapshot doc contexte                                                  | À trier post-event                      |

## Front — `attendee-ems-front`

| Branche                | État (14/07)                                                             | Action                 |
| ---------------------- | ------------------------------------------------------------------------ | ---------------------- |
| `main`                 | = staging (strictement identique) · prod déployée dessus                 | —                      |
| `staging`              | Base d'intégration · CD staging auto (build `VITE_API_BASE_URL` staging) | —                      |
| `chore/ci-cd`          | ~~Entièrement mergée~~                                                   | 🗑️ **Supprimée 14/07** |
| `dev`                  | 14 commits en retard sur main, 0 d'avance — vestige                      | Supprimable            |
| `gcp-migration-stable` | 12 en retard, 0 d'avance — snapshot                                      | Supprimable            |

> Note front : le travail H de Corentin (2 commits `feat(sessions)` du 13/07) a été **commité
> directement sur `staging`** (pas de branche) et mergé dans `main` le 13/07 au soir → en prod.

## Mobile — `attendee-ems-mobile` (vérifié 2026-07-14)

> Pas de `staging` sur ce repo : circuit `feature → main`, releases via branches `release/x.y.z`.

| Branche                               | État (14/07)                                                                                      | Action                           |
| ------------------------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------- |
| `main`                                | `759fc75` = merge **PR #9 `feat/qr-hmac-security`** (13/07) · `app.json` = 1.1.9 (non releasée)   | Base du prochain chantier mobile |
| `feat/qr-hmac-security`               | Entièrement mergée dans main (PR #9)                                                              | 🗑️ Supprimable                   |
| `release/1.1.7`                       | Dernière release (03/06) — 20 commits de retard sur main, **SANS le QR HMAC**                     | Version actuellement diffusée    |
| `release/1.1.6` / `release/1.1.5`     | Anciennes releases (03/06) — ⚠️ local `release/1.1.6` désaligné d'origin (`c2b39b7` vs `fe146d4`) | Garder (historique releases)     |
| `workstream/diagnostic-stabilisation` | 8 commits de retard, 0 d'avance (dernier merge 18/06)                                             | Supprimable (tout est dans main) |
| `context-organization`                | Snapshot doc contexte (22/06)                                                                     | À trier post-event               |

> ⚠️ Artefacts non versionnables à la racine du repo : `attendee-ems-mobile.ipa`, `temp_bundle.hbc`, `temp_bundle.js`.

### 📌 Décision 2026-07-14 — release mobile en STAND-BY

- **Où vit le code QR HMAC mobile :** sur **`main`** (mergé via PR #9 le 13/07). Aucune release ne le contient
  (dernière diffusée = 1.1.7 du 03/06).
- **Décision :** **pas de release maintenant.** Un **chantier mobile** est à venir ; la release se fera **à la fin
  de ce chantier** et embarquera d'un coup : QR HMAC + les adaptations exigées par les changements back récents
  - les éventuelles modifs back à venir qui impacteront le mobile.
- **Pourquoi :** éviter des releases mobiles successives (store/EAS coûteux en cycle) alors que d'autres changements
  back nécessitant des modifs mobile sont probables à court terme.
- **Conséquence (corrigée après vérif code du 14/07) :** la 1.1.7 reste compatible avec le back HMAC en prod —
  elle relaie la chaîne scannée telle quelle et les QR des emails sont re-signés côté serveur à l'affichage.
  Le back est **déjà strict** (UUID brut refusé). ⚠️ Seul risque : les **supports figés d'avant le 13/07**
  (badges imprimés, PDF stockés, images QR cachées) sont rejetés → confirmer qu'aucun ne servira pour l'event.

---

## Historique des opérations

| Date       | Opération                                                                                                                                         |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-07-13 | Back : cherry-pick prettier `8b53222` → staging (`0aa540f`) · merge staging → main (`b50339a`)                                                    |
| 2026-07-13 | Front : merge `chore/ci-cd` → staging (`c274cf4`, 2 conflits sessions résolus) · ff staging → main                                                |
| 2026-07-13 | Fixes CI/CD mergés au fil de l'eau (vitest exclude, permissions actions:read, healthchecks, nginx reload) — staging et main maintenus alignés     |
| 2026-07-13 | Rangement `scripts/` + `test/` back (`72d97f6`) — validé par CD staging auto                                                                      |
| 2026-07-14 | 🗑️ Suppression `chore/ci-cd` (back + front) · réalignement gotenberg = staging                                                                    |
| 2026-07-14 | Back : création `chore/monitoring` depuis staging (`72d97f6`) — finalisation 0-MON + rangement tests éventuel                                     |
| 2026-07-14 | Mobile : état des lieux ajouté · décision **release mobile en stand-by** (attendre fin du chantier mobile à venir)                                |
| 2026-07-14 | Back : merge PR #20 (`feat/c2-mailgun-integration`) vers `staging` puis suppression de branche · merge `staging` → `chore/monitoring` (`c426a7a`) |
| 2026-07-16 | Back : `feat/internal-mailing-wave1` vérifiée sans diff vs `staging` · `chore/monitoring` mergée dans `staging` (`63c2bd6`) pour livrer 0-MON     |
