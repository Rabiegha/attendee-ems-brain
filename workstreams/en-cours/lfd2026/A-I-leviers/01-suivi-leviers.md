# 01 — Suivi des leviers (chantier A refonte) — LFD 2026

> **À QUOI SERT CE FICHIER :** c'est le **tableau de bord vivant** de la refonte propre (chantier A).
> Source de vérité de **ce qui est fait / ce qui reste**, levier par levier, branche par branche.
> **À ouvrir au début de chaque nouveau chat** (on fait 1 chat par levier) et sur n'importe quel poste.
>
> - Le **quoi/pourquoi** (cadrage, temps estimés) → [00-plan-action.md](../00-plan-action.md) §0–§2.
> - Le **focus global** (tous chantiers) → [NOW.md](../../../../workspace-rabie/NOW.md).
> - Les **learnings** produits → [learnings](../../../../learnings/README.md).
> - Les **leviers éventuels si L9b/L9a/L9.1 ne suffisent pas** → [leviers-eventuels-capacite.md](./leviers-eventuels-capacite.md).
>
> **Principe maître :** _ne rien perdre, tout rejouer proprement_. 1 levier = 1 branche = 1 commit
> (diff minimal) = 1 PR = 1 mesure avant/après. Archive de référence : branche `staging-archive-2026-06-25`
>
> - tag `archive/staging-2026-06-25` (back & brain).
>
> **Dernière mise à jour :** 2026-07-20

---

## Conventions à respecter (rappel, pour tout nouveau chat)

- ✅ **Pas de doc infra dans `attendee-ems-back`** → toute doc va dans `attendee-ems-brain`.
- ✅ **Format-on-save OFF** pendant tout le chantier (jamais `npm run format` / Format Document) → diffs chirurgicaux.
- ✅ **1 levier = 1 commit propre** sur sa **branche dédiée**.
- ✅ **Workflow** (décision 2026-07-10, rappel renforcé 2026-07-17) : branche dédiée → test **local** → **MR vers `staging`**
  (branche d'environnement : le serveur staging est construit **depuis `staging`**) → rebuild
  api sur le VPS + campagne **k6** → en fin de chantier, **PR contrôlée `staging` → `main`**.
  `staging` = espace de validation, `main` = production. Aucun levier ne doit cibler `main` directement.
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
| **L1/L2/L10** stack staging + pgBouncer + pool                   | `infra/staging-stack`          | infra/config                             | 🟣 **Sur `staging` + mesuré** (2026-07-10) | [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) **mergée sur `staging`** | ✅ avant/après fait → [détail ci-dessous](#-l1l2l10--stack-staging--pgbouncer--pool) |
| **L9a** transaction event allégée (`registerToEvent`)             | `feat/register-event-tx-slim`  | **code métier** (chirurgie)              | 🟣 Sur `staging` + mesuré en intégration | [#39](https://github.com/Rabiegha/attendee-ems-back/pull/39) + [#42](https://github.com/Rabiegha/attendee-ems-back/pull/42) mergées | ✅ compteur event actif pendant la campagne 70/s |
| **L9b** transaction session allégée (`registerToSession`)         | `feat/register-session-tx-slim` + correctifs dédiés | **code métier** (chirurgie, priorité LFD) | 🟣 Sur `staging` + campagne finale | [#38](https://github.com/Rabiegha/attendee-ems-back/pull/38), [#45](https://github.com/Rabiegha/attendee-ems-back/pull/45), [#50](https://github.com/Rabiegha/attendee-ems-back/pull/50) mergées | ✅ 70/s soutenues et 250 simultanées ; 75/s/350 refusées |
| **L9.1** compteur présence session O(1) (candidat, **priorisé**) | `feat/session-present-counter` | code métier (colonne PG `present_count`) | 🟣 Mergé, déployé et régressions vertes | [#40](https://github.com/Rabiegha/attendee-ems-back/pull/40) mergée sur `staging` | ⏳ charge check-in dédiée encore à exécuter |
| **L7** email async BullMQ                                        | couvert par C2 (`email.send`)  | audit de non-duplication                  | ✅ Déjà livré par C2                     | [#20](https://github.com/Rabiegha/attendee-ems-back/pull/20)                           | ✅ hors hot path ESP ; queue active staging                                           |
| **L8** worker `PROCESS_ROLE` (Voie A)                            | `feat/process-role-worker`     | code + topologie/CD                      | 🟢 Codé/testé, PR bloquée par billing CI | [#55](https://github.com/Rabiegha/attendee-ems-back/pull/55) ouverte                   | ⏳ après CI/CD et fenêtre de charge                                                   |
| **L3** `directUrl` Prisma                                        | `chore/prisma-directurl`       | config (+5 lignes)                       | 🟣 Sur `staging`                         | [#32](https://github.com/Rabiegha/attendee-ems-back/pull/32) **mergée sur `staging`** | n/a                                                                                  |

**Légende statut :** ⚪ à faire · 🟡 en cours · 🟢 codé+testé local · 🔵 sur `dev` · 🟣 sur `staging`+mesuré · ✅ clos.

---

## Étapes transverses du chantier A

| #   | Étape                                            | Statut               | Note                                                                                                                                                                                                                 |
| --- | ------------------------------------------------ | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0   | Geler & archiver                                 | ✅ Fait              | tag `archive/staging-2026-06-25` + branche `staging-archive-2026-06-25` (back). Brain : à vérifier.                                                                                                                  |
| 1   | Séparer reformatage / logique                    | ✅ Fait              | format-on-save vérifié OFF (learning créé).                                                                                                                                                                          |
| 2   | Rejouer les leviers (chantier A + L9.1)          | 🟡 En cours          | 7/7 couverts ou codés — six livrés sur `staging`, L8 en PR #55 avec 116/116 unitaires mais GitHub refuse les runners pour billing. Campagne inscription finale : 70/s soutenues et 250 simultanées validées. Reste : CI/CD L8 et charge check-in L9.1. |
| 3   | Réconcilier la doc (~25/s CPU vs ~33/s DB)       | ⚪ À faire           | dans `infra-scaling-pca/README.md`.                                                                                                                                                                                  |
| 4   | Post-mortem 502 prod                             | ✅ Fait (2026-07-10) | [bugs/fait/2026-06-25-prod-502-collision-compose.md](../../../../bugs/fait/2026-06-25-prod-502-collision-compose.md) + garde-fous suivis dans [garde-fous-deploiement-staging.md](./garde-fous-deploiement-staging.md). |
| 5   | Isoler le compose prod (PR dédiée)               | ⚪ À faire           | `docker-compose.prod.yml`, ne pas merger sans revue.                                                                                                                                                                 |
| 6   | Cadrer l'interdit (L13 cluster, L12 sync_commit) | ⚪ À faire           | note, pas de code.                                                                                                                                                                                                   |

### Note capacite — plans B

Le chemin nominal devient **L9b puis L9a puis L9.1 puis mesure k6**. L9b est prioritaire car
`back-office-event` utilise `POST /api/public/events/sessions/:sessionToken/register`. Si le plafond reste trop bas, les leviers a
considerer sont documentes dans [leviers-eventuels-capacite.md](./leviers-eventuels-capacite.md), avec
une option de paquet livrable en 10 jours : portier Redis, cache public, queues/mode degrade, puis test
combine.

### Note branche / prod — 2026-07-17

- L3 et L9b sont sur `staging` et déployés staging.
- Une PR globale `staging -> main` (#34) a révélé une divergence : `main` contenait aussi des commits absents de `staging`.
- #34 a été fermée et remplacée par [#35](https://github.com/Rabiegha/attendee-ems-back/pull/35), branche dédiée `chore/reconcile-staging-main`, conflits résolus localement, CI verte.
- Ne pas merger #35 sans décision explicite : `main` peut déclencher la chaîne prod.

---

## Détail par levier

### 🟣 L1/L2/L10 — stack staging + pgBouncer + pool

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
- **Learnings liés :** [stack staging](../../../../learnings/2026-07-07-stack-staging-concepts-infra.md) · [pgBouncer/pool](../../../../learnings/2026-07-07-pgbouncer-et-pool-db.md) · [directUrl](../../../../learnings/2026-07-07-directurl-prisma.md) · [git checkout/reset](../../../../learnings/2026-07-07-git-checkout-ref-fichier.md).

### 🟣 L9a — transaction d'inscription event allégée

- **Endpoint :** `POST /api/public/events/:publicToken/register`.
- **Méthode :** `PublicService.registerToEvent`.
- **Branche :** `feat/register-event-tx-slim` · **Commit :** `873a08f` · **PR :** [#39](https://github.com/Rabiegha/attendee-ems-back/pull/39) mergée · **Merge staging :** `4253d399`.
- **Problème :** `registration.count` capacité event dans la transaction + upsert attendee + create/restore registration.
- **Nature :** ⚠️ **vraie chirurgie de code** → **test de non-régression obligatoire** (inscriptions identiques, pas de doublon, quota respecté).
- **Source de rejeu :** archive `staging-archive-2026-06-25` (rejouer à la main, pas cherry-pick du fourre-tout).
- **Livré :** compteur event atomique `approved_registration_count`, transaction publique raccourcie,
  garde commune aux writers admin/public/import, maintenance scoped et **78/78 E2E** au moment de la PR.
- **Déploiement :** CI PR, CI `staging` et CD verts ; health externe sert le SHA exact
  `4253d3993670f8714c43c070dcbeaef967370166`.
- **Détail :** [learning L9a](./05-learning-l9a-approved-registration-count.md).
- **Mesuré en intégration :** le compteur événement L9.a est actif pendant toute la campagne session
  finale : 70 inscriptions/s pendant deux minutes sans erreur ni dérive. Le test discriminant avec
  auto-approve coupé a confirmé son rôle dans l'ancien goulot, puis la PR #50 a raccourci la section
  critique L9.b.
- **Reste :** scénario k6 dédié au endpoint event `registerToEvent` si ce flux est utilisé par LFD.

### 🟣 L9b — transaction d'inscription session allégée

- **Endpoint :** `POST /api/public/events/sessions/:sessionToken/register` (**utilisé par `back-office-event` LFD**).
- **Méthode :** `PublicService.registerToSession`.
- **Branche :** `feat/register-session-tx-slim` · **MR :** [#33](https://github.com/Rabiegha/attendee-ems-back/pull/33) mergée sur `staging` · **Merge staging :** `44a9ff4`.
- **Fichiers principaux :** `prisma/schema.prisma`, migration `20260717170000_session_registered_count`, `public.service.ts`, `registrations.service.ts`, `sessions.service.ts`, `test/sessions-h.e2e-spec.ts`.
- **Fait :** compteur `sessions.registered_count` + backfill, capacité session lue sous verrou `FOR UPDATE`, incrément O(1) sur inscription publique session, refresh du compteur sur chemins admin/public/remove, stats live basées sur le compteur.
- **Validation initiale :** PR #33 CI verte, merge `staging`, CI push `staging` verte, CD staging vert, health staging OK (`version=44a9ff4ffc3cac590801b1efe17069e095b9fac1`).
- **Note :** L9b ne remplace pas L9a. Il ajoute le hot path réellement utilisé pour LFD.
- **Durcissements livrés :** réservation tardive PR #45 puis réservation + création du choix en une
  instruction SQL atomique PR #50. CI PR/push, CD, 82 unitaires, 54 E2E ciblés et build verts.
- **Mesure finale :** 70/s pendant 120 s = 8 400/8 400, p95 223 ms, p99 405 ms, zéro
  erreur/perte ; choc 250 = 250/250, p95 498 ms. 75/s soutenues et 350 simultanées échouent.
- **Rapport :** [campagne L9.b atomique](../session-travail-autonome/rapports-load-tests/2026-07-20-1635-1708-l9b-atomique-campagne-finale.md).
- **Reste :** ne pas revendiquer 3 000 simultanées sans orchestrateur multi-générateur et preuve
  distribuée ; rejouer après toute modification réelle du hot path/audit synchrone/pool DB.

### 🟣 L9.1 — compteur de présence session O(1)

- **Branche :** `feat/session-present-counter` · **Commit :** `ad80b3c` · **PR :** [#40](https://github.com/Rabiegha/attendee-ems-back/pull/40) mergée · **Merge staging :** `c75febd`.
- **Problème performance :** `2× COUNT(*)` sur `session_scans` à chaque scan IN → **O(n)**.
- **Problème correction confirmé le 19/07 :** ces `COUNT` sont exécutés avant la transaction et le
  verrou actuel est par `sessionId:registrationId`. Deux participants différents scannés en même temps
  peuvent observer la dernière place libre et être admis tous les deux.
- **Levier :** compteur PostgreSQL `present_count` O(1) mis à jour atomiquement dans la même
  transaction que le scan, avec garde `present_count < checkin_capacity` et sérialisation au niveau
  session. Redis reste un dérivé éventuel pour l'affichage, pas la source de vérité.
- **Consommateur CDC :** [J-ENTREES](../J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd)
  lit ce compteur pour les présents actuels et le taux de remplissage physique. Le total post-event
  « entré au moins une fois » reste une métrique distincte fondée sur les `IN admitted`.
- **Nature :** ⚠️ **code métier** → tests H-T20→H-T25 obligatoires : concurrence/replay, cohérence
  IN/OUT, capacité physique et statut présent/absent.
- **Conditionné mesure** à l'origine, **priorisé** pour ce cycle (décision 2026-07-08).
- **Détail conceptuel :** [workstream 02 §2g](../../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md).
- **Livré sur staging :** migration/backfill depuis le dernier scan admis, compteur atomique IN/OUT,
  garde de dernière place, contrat `duplicate: true|false`, undo/cascades, maintenance scoped et
  **9/9 ciblés, 52/52 régressions L9/H, 87/87 E2E complets, 82/82 unitaires**.
- **Détail :** [learning L9.1](./06-learning-l91-session-present-count.md).
- **Reste :** mesure k6 check-in dédiée et intégration/recette du contrat dans M/J.

### ✅ L7 — email async (BullMQ), déjà couvert par C2

- **Décision après audit du 20/07 :** ne pas créer une seconde implémentation L7. Le chantier C2 a
  déjà livré la queue BullMQ `email.send`, le `jobId` idempotent, les retries/backoff, le throttle
  Redis à chaud, le processor Mailgun/SMTP et les événements de délivrabilité.
- **Chemin inscription :** l'appel post-commit prépare/enfile le job ; avec
  `EMAIL_QUEUE_ENABLED=true` en staging, l'API n'attend pas le réseau de l'ESP.
- **PR source :** [#20](https://github.com/Rabiegha/attendee-ems-back/pull/20).
- **Reste hors L7 :** séparer le processor du process API appartient à L8 ; orchestrer PDF puis
  email de session appartient à C2.1.

### 🟢 L8 — worker `PROCESS_ROLE` (Voie A), PR ouverte

- **Branche :** `feat/process-role-worker` · **Nature :** gating dans `main.ts`.
- **But :** pouvoir lancer un process dédié « worker » (consomme la file) distinct de l'API.
- **Implémentation rejouée manuellement :** `api` sert HTTP/produit les jobs sans processor ;
  `worker` consomme email/export sans port HTTP et sans migrations ; `all` conserve le comportement
  historique. Valeur invalide fail-fast.
- **Topologie :** même image SHA pour API/worker staging, health worker et rollback vers un ancien
  compose sans worker gérés par le script CD.
- **Validation locale :** **116/116 unitaires**, tests des métadonnées Nest pour les trois rôles,
  build, TypeScript, ESLint et ShellCheck verts.
- **PR :** [#55](https://github.com/Rabiegha/attendee-ems-back/pull/55), non mergée. Les quatre jobs
  sont refusés avant exécution par le blocage Billing & plans GitHub Actions ; ne pas merger avant
  rétablissement et CI verte.
- **Reste :** CI, merge/CD, worker healthy puis smoke enqueue→consume ; charge seulement dans une
  nouvelle fenêtre autorisée.

### 🟣 L3 — `directUrl` Prisma

- **Branche :** `chore/prisma-directurl` · **Nature :** +5 lignes dans `datasource` de `schema.prisma`.
- **But :** URL directe (port 5432) pour les migrations, l'app passant par pgBouncer (6432).
- **⚠️ Urgence montée d'un cran (2026-07-10) :** sans `directUrl`, `prisma migrate deploy` via pgBouncer
  **crash-loop en P1002** (advisory lock) — vécu sur staging pendant la campagne k6. Workaround actuel :
  `RUN_MIGRATIONS=false` dans `.env.staging` (les migrations doivent être lancées à la main en direct).
- **Statut 2026-07-17 :** mergé sur `staging` via [#32](https://github.com/Rabiegha/attendee-ems-back/pull/32), puis repris dans la PR de réconciliation [#35](https://github.com/Rabiegha/attendee-ems-back/pull/35) vers `main`.
- **Reste :** surveiller la prochaine migration staging/prod avec `DIRECT_URL` effectif.

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
