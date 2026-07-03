# LFD 2026 — HANDOFF de session (passage de machine) — 2026-06-25

> **But de ce document** : permettre de **reprendre le travail sur une autre machine** sans
> rien perdre. Il récapitule le déroulé de la session, ce qui a été fait, ce qui reste, les
> chiffres mesurés, les fichiers touchés, et les règles critiques.
> La mémoire interne de l'assistant **ne suit pas** d'une machine à l'autre → **tout est ici**.
>
> Lecture associée (écrits pendant cette session) :
>
> - [lfd-2026-load-test-results.md](./lfd-2026-load-test-results.md) — rapport technique (résultats réels)
> - [lfd-2026-reproduction.md](./lfd-2026-reproduction.md) — refaire les optimisations + tests soi-même
> - [lfd-2026-client-report.md](./lfd-2026-client-report.md) — rapport client MEAE (pré-vente)
> - Workstreams liés : [async-badge-email](../workstreams/async-badge-email/README.md), [api-scaling-clustering](../workstreams/api-scaling-clustering/README.md)

---

## 1. Contexte & objectif

Préparer l'API EMS pour le contrat **MEAE « La Fabrique de la Diplomatie 2026 »** (4–5 sept 2026,
~20 000 visiteurs sur 2 jours, **pic contractuel de 3 000 inscriptions simultanées**, CDC § 2.3 / § 6.1).
Échéance : ~3 mois.

Demande initiale : un **rapport de capacité honnête** (vérité du réel, pas de chiffres inventés),
puis optimiser, puis fournir : (1) rapport crash-test après modifs, (2) doc de reproduction pour
refaire sur prod, (3) rapport client avec résultats de simulation, (4) pousser les tests le plus
loin possible.

---

## 2. Déroulé de la session (chronologique)

1. **Diagnostic initial** : l'API est un **seul process Node** (NestJS, pas de cluster) →
   suspicion de goulot. Email post-inscription = `setImmediate` in-process (non durable),
   badge = hors flux d'inscription (généré à l'impression, c'est un **QR code** dans l'email,
   pas un PDF).
2. **Mise en place staging isolée** sur le VPS (stack `ems-staging`, données anonymisées RGPD,
   Mailpit). Cf. [staging-setup.md](./staging-setup.md).
3. **Leviers appliqués et mesurés** (chacun avec un test de charge k6 50→400 VUs) :
   - **#1** pool DB 100 + pgBouncer → erreurs 14,9 % → 10,4 % mais latence pire.
   - **#2** email → BullMQ (async, durable, retry) → pas de gain débit (l'enqueue était dans la transaction).
   - **#3** transaction d'inscription allégée (email après commit, logs retirés) → pas de gain.
4. **Diagnostic décisif** (test charge fixe 250 VUs + échantillonnage) : **DB pas saturée**
   (1–4 connexions actives /100), **CPU au taquet** (220–404 %/400 %). → le mur était le
   **CPU / event loop du process unique**, pas la DB.
5. **Voie B (cluster)** testée sur staging (éphémère) : 3 instances → débit **+12 %**, erreurs
   **−37 %**, CPU réparti. Mais x3 process ≠ x3 débit → un **2ᵉ goulot partagé** apparaît.
6. **Voie A (worker email/QR séparé)** codée + testée : **aucun gain débit** (worker idle ~8 %).
   → le rendu email/QR n'est **pas** le coût CPU dominant.
7. **Test de confirmation 4 cœurs** : **32,65 reg/s** = identique aux 2 cœurs, malgré **367 % CPU**.
   → **PREUVE FINALE** : le plafond (~33–37 reg/s) n'est **ni le CPU, ni le pool, ni le nombre de
   process** → c'est une **sérialisation sur l'écriture DB** (transaction d'inscription).
8. **Teardown complet** des conteneurs de test, **prod vérifiée `200`** tout du long.
9. **Rédaction** des 3 livrables (résultats, reproduction, client).

---

## 3. Résultats mesurés (réels, plancher conservateur)

| Configuration                | Inscriptions/s | Échec HTTP |       p95 | CPU                        |
| ---------------------------- | -------------: | ---------: | --------: | -------------------------- |
| 1 process · 2 cœurs          |           32,8 |     12,7 % |    3,38 s | 156–206 % / 200 %          |
| **1 process · 4 cœurs**      |      **32,65** |     12,6 % |    3,30 s | **327–367 % / 400 %**      |
| Voie A (API + worker séparé) |           32,3 |     12,9 % |    3,32 s | API 170 % · worker **8 %** |
| **Cluster 3 × 2 cœurs**      |       **36,8** |  **8,0 %** |    3,29 s | réparti, non saturé        |
| Référence saine ≤ 100 VUs    |              — |    **0 %** | **0,2 s** | nominal                    |

- **Plafond système ≈ 33–37 reg/s** = sérialisation écriture DB.
- **Point de rupture ≈ 150–200 VUs** (≈ 500–650 tentatives/s).
- **Connexions DB : 1–4 / 100** au pic (pool jamais le problème).

> ⚠️ **Caveat honnêteté** : k6 tournait sur le **même hôte 8 cœurs** que l'API + DB →
> chiffres = **plancher**. Une mesure propre (k6 externe) donnera mieux. À refaire à J‑7.

**Interprétation contractuelle** : 33 reg/s = ~2 000/min → 3 000 absorbées en **~90 s**.
Arrivée réaliste 20k/2j ≈ **5 reg/s**. Large marge.

---

## 4. Fichiers touchés (à committer / poussés sur `staging`)

### Code (back)

| Fichier                                      | Nature      | Rôle                                      |
| -------------------------------------------- | ----------- | ----------------------------------------- |
| `src/infra/queue/queue-names.ts`             | modifié     | + `EMAILS_REGISTRATION`                   |
| `src/infra/queue/index.ts`                   | modifié     | export helpers `process-role`             |
| `src/infra/queue/process-role.ts`            | **nouveau** | `PROCESS_ROLE` (api/worker/all)           |
| `src/main.ts`                                | modifié     | mode worker (`app.init()` si pas HTTP)    |
| `src/modules/email/email-queue.types.ts`     | **nouveau** | types jobs email                          |
| `src/modules/email/email-queue.service.ts`   | **nouveau** | producteur (enqueue, fire-and-forget)     |
| `src/modules/email/email-queue.processor.ts` | **nouveau** | worker email/QR (concurrency bornée)      |
| `src/modules/email/email.module.ts`          | modifié     | enregistre queue + gating worker par rôle |
| `src/modules/exports/exports.module.ts`      | modifié     | gating workers exports par rôle           |
| `src/modules/public/public.service.ts`       | modifié     | transaction allégée + email après commit  |
| `docker-compose.staging.yml`                 | modifié     | pool pgBouncer 100, cpus/mem (Levier #1)  |
| `scripts/k6/register-stress.js`              | (existant)  | rampe 50→400 VUs                          |
| `scripts/k6/register-fixed.js`               | **nouveau** | charge fixe 250 VUs (diagnostic)          |

### Docs (back/docs/infra)

- `lfd-2026-load-test-results.md`, `lfd-2026-reproduction.md`, `lfd-2026-client-report.md`,
  ce handoff.

> **Rétro-compatibilité** : sans `PROCESS_ROLE`, le défaut est `all` → comportement
> **strictement identique** à avant. La séparation worker est **opt-in**.

> **Build validé** : `nest build` dans Docker compile proprement (l'image `ems-staging-api` a
> été rebuildée avec succès). Les erreurs `tsc` en local sont **environnementales**
> (`node_modules` incomplet : `bullmq` manquant localement) — pas des erreurs réelles.

---

## 5. CE QUI RESTE À FAIRE (priorisé)

| Prio | Tâche                                       | Détail                                                                                                                                                                                                                                          | Risque                   |
| ---- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1    | **Optimiser la transaction d'écriture**     | Sortir le `registration.count(...)` de capacité du chemin chaud (compteur dénormalisé / verrou ciblé) + raccourcir la transaction au strict `create`. **C'est LE levier** pour dépasser 37 reg/s. Préserver l'anti-surbooking. Voir results §5. | 🟠 Moyen (métier jauge)  |
| 2    | **Mesure propre k6**                        | Relancer depuis une **machine externe** (pas le VPS) pour des chiffres réels non-plancher.                                                                                                                                                      | Faible                   |
| 3    | **Déployer Voie A en prod**                 | API `PROCESS_ROLE=api` + 1 conteneur worker `PROCESS_ROLE=worker` (`RUN_MIGRATIONS=false`). Durabilité email + protection event loop si SMTP lent.                                                                                              | Faible                   |
| 4    | **Spike test 3000 VUs**                     | « Thundering herd » pour démontrer la capacité max (demandé, non encore fait).                                                                                                                                                                  | Faible (staging)         |
| 5    | **Voie B (cluster) — PAS avant clustering** | Nécessite redis-adapter + sticky + présence Redis (workstream `api-scaling-clustering`) sinon **casse l'impression silencieusement**.                                                                                                           | 🔴 Élevé sans pré-requis |
| 6    | **Aligner executive-summary**               | Corriger « badge **PDF** dans l'email » → en réalité **QR code**.                                                                                                                                                                               | Faible                   |
| 7    | **Finaliser workstream async-badge-email**  | Levier #2 (email BullMQ) **partiellement fait** ; badge → BullMQ reste à faire.                                                                                                                                                                 | Moyen                    |

---

## 6. RÈGLES CRITIQUES (à respecter absolument)

- 🚨 **Voie B (cluster) = TEST/STAGING UNIQUEMENT, JAMAIS EN PROD** tant que le workstream
  clustering n'est pas livré. Le multi-instances **casse l'impression temps réel
  silencieusement** (registres présence/imprimantes en mémoire de process). Ne jamais ajouter
  de replicas à `docker-compose.prod.yml`.
- **Conteneurs de test = éphémères `ems-loadtest-*`**, à détruire après (`docker rm -f`).
  Toujours vérifier ensuite : `docker ps -a | grep loadtest` doit être vide.
- **Toujours `--env-file .env.staging`** pour la stack staging, et un `name:` distinct
  (`ems-staging`) pour ne pas écraser la prod.
- **Vérifier la prod après chaque opération** : `curl -sk https://127.0.0.1/api/health` = `200`.
- **Jamais `down -v`** sur staging (détruit les volumes).
- **Honnêteté totale** dans les rapports : chiffres mesurés, pas inventés.
- **Sécurité** : ne jamais committer de secret. **Régénérer tout secret exposé** (cf. DEBTS
  ems-brain : clé PrintNode, et tout mot de passe ayant transité).

---

## 7. Infos opérationnelles (pour reprendre sur l'autre machine)

| Élément             | Valeur                                                                                                                                        |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| VPS prod+staging    | OVH `51.75.252.74`, user `debian`, code dans `/opt/ems-attendee/backend/`                                                                     |
| Accès SSH           | clé `~/.ssh/ems_staging_ed25519` (recopier la clé sur la nouvelle machine)                                                                    |
| Hôte                | 8 cœurs, 22 Gi RAM                                                                                                                            |
| Stack staging       | `ems-staging-api` (3001), `ems-staging-postgres` (5433), `ems-staging-pgbouncer` (6432), `ems-staging-redis`, `ems-staging-mailpit` (UI 8025) |
| DB staging          | user `ems_staging`, db `ems_staging`                                                                                                          |
| DB prod             | user `ems_prod`, db `ems_production`                                                                                                          |
| Branche back        | `staging` (le VPS, lui, tourne sur `main` — déploiement = copie de fichiers, pas `git pull`)                                                  |
| Token event de test | `dGA2GXHnopydhx3E`                                                                                                                            |

> ⚠️ Le VPS déploie par **copie de fichiers** (scp) + rebuild image, **pas** par `git pull`.
> Voir [lfd-2026-reproduction.md](./lfd-2026-reproduction.md) pour la procédure exacte.

---

## 8. Points de vigilance laissés ouverts

- **Image `ems-staging-api` rebuildée avec Voie A**, mais le conteneur `ems-staging-api` tourne
  encore l'ancienne image (sans `PROCESS_ROLE` = `all` = identique). Aucun risque ; recréer le
  conteneur si on veut la nouvelle image active.
- **pgBouncer prod** : correctif `LISTEN_PORT 6432` **non déployé** en prod (flagué).
- **Containers `unhealthy`** (`ems-api`, `ems-staging-api`) renvoient quand même `200` —
  faux-positif healthcheck connu.
- **Commits non poussés** : le travail de cette session est commité sur `staging` et poussé
  dans le cadre de ce handoff. Le VPS n'est **pas** mis à jour automatiquement.
