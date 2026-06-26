# Audit de la branche `staging` — travaux scaling LFD 2026

> **Objet :** tracer **tout** ce qui a été fait sur la branche `staging` (repos
> `attendee-ems-back` et `attendee-ems-brain`) depuis mercredi soir, lister **tous** les
> leviers, leurs exécutions et résultats réels, puis poser un **plan d'annulation +
> refonte propre** en workstreams.
> **Date de l'audit :** 26 juin 2026
> **Branches concernées :** `attendee-ems-back@staging`, `attendee-ems-brain@staging`
> **Règle absolue rappelée :** ❌ ne jamais toucher la production. Toutes les opérations ont
> été menées sur la stack isolée `ems-staging`.

---

## 0. TL;DR

- **38 fichiers** touchés côté back (+4 222 / −850 lignes), **5 fichiers** côté brain.
- **10 commits back** + **3 commits brain**, tous entre le **25/06 01:08** et le **25/06 18:07**
  (le travail a démarré dans la nuit de mercredi à jeudi).
- **14 leviers** identifiés (infra + code + tests), dont **3 déployés sur staging**, **0 en
  prod**, **1 testé puis abandonné**, **1 interdit en prod**.
- **4 campagnes de test** exécutées sur staging, prod surveillée et **intègre** (`health=200`,
  CPU ≈ 0 %) tout du long.
- **2 problèmes de propreté majeurs** : (a) un commit fourre-tout `feat(scaling)` qui mélange
  4 leviers + 3 rapports + un reformatage complet de fichiers ; (b) des chiffres incohérents
  entre les docs brain (« ~33/s, plafond = DB ») et les mesures réelles (« ~25/s, plafond =
  CPU »).
- **1 incident prod** survenu et corrigé pendant la session (collision de nom de projet Docker
  → 502 prod), à documenter comme post-mortem.

---

## 1. Chronologie des commits

### `attendee-ems-back` (branche `staging`)

| Heure (25/06) | Commit | Sujet |
| --- | --- | --- |
| 01:08 | `c7247ae` | infra scaling LFD 2026 (pgbouncer, stack staging, dump/restore, docs async badge+email) |
| 01:12 | `bd858f9` | fix(pgbouncer) : `LISTEN_PORT=6432` + healthcheck `pg_isready` |
| 01:15 | `1ba886f` | staging : script anonymisation PII (RGPD) |
| 02:13 | `8afdd12` | staging : script k6 baseline flux inscription |
| 02:19 | `5f72daa` | **fix(staging) : `name=ems-staging` pour isoler du projet prod — incident prod 502** |
| 02:30 | `0da71f0` | staging : script k6 stress (paliers 50→400 VUs) |
| 02:46 | `47c82bf` | fix(email) : auth SMTP conditionnelle (Mailpit sans auth) |
| 11:35 | `d3f3421` | **feat(scaling) : email async BullMQ + worker `PROCESS_ROLE` + transaction allégée + 3 rapports & handoff** |
| 11:38 | `98a678e` | session handoff |
| 18:07 | `1d2978b` | rapport |

### `attendee-ems-brain` (branche `staging`)

| Heure (25/06) | Commit | Sujet |
| --- | --- | --- |
| 01:09 | `ab99893` | backlog tech LFD 2026 (bullmq, scaling pic inscription, puppeteer) |
| 11:35 | `f0c98c4` | workstream `api-scaling-lfd2026` + MAJ NOW/INBOX/DEBTS |
| 11:38 | `a39beae` | Update DEBTS.md |

---

## 2. Inventaire complet des fichiers

### 2.1 `attendee-ems-back` — 38 fichiers (vs `main`)

**Infra / environnement (nouveau)**
- `docker-compose.staging.yml` — stack staging isolée
- `.env.staging.example` — variables staging
- `scripts/db-dump.sh` — dump/restore prod→staging
- `scripts/staging-anonymize.sql` — anonymisation PII (RGPD)
- `scripts/staging-make-env.sh` — génération `.env` staging

**Infra / environnement (modifié)**
- `docker-compose.prod.yml` (+52) — ⚠️ touche le compose **prod** (non déployé, à revoir)
- `nginx/nginx.conf` (+11)
- `.gitignore` (+1)

**Code — hot path & queue**
- `src/modules/public/public.service.ts` (+517/−381) — transaction d'inscription allégée
- `src/main.ts` (+88) — gating `PROCESS_ROLE` (Voie A) + reformatage
- `src/infra/queue/process-role.ts` (nouveau) — rôle de process api/worker/all
- `src/infra/queue/index.ts`, `queue-names.ts` (modifiés)
- `src/modules/email/email-queue.{processor,service,types}.ts` (nouveaux) — email async BullMQ
- `src/modules/email/email.{module,service}.ts` (modifiés)
- `src/modules/exports/exports.module.ts` (modifié)
- `prisma/schema.prisma` (+5) — `directUrl` (migrations compatibles pgBouncer)

**Tests de charge — `scripts/k6/`**
- `crash-test-full.js`, `endurance-reliability.js`, `spike-absorption.js`,
  `register-baseline.js`, `register-fixed.js`, `register-stress.js`

**Documentation — `docs/infra/`**
- `2026-06-09-crash-test-annuel.md`, `2026-06-24-rapport-client-fiabilite-pics.md`
- `lfd-2026-client-report.md`, `lfd-2026-load-test-results.md`, `lfd-2026-load-test-plan.md`,
  `lfd-2026-reproduction.md`, `lfd-2026-db-write-lever-results.md`,
  `lfd-2026-session-handoff-2026-06-25.md`, `staging-setup.md`
- `docs/workstreams/api-scaling-clustering/README.md`, `docs/workstreams/async-badge-email/README.md`
- `docs/other/migrations and refactoring/migration-GCP/ANALYSE_CAPACITE_INFRASTRUCTURE.md` (modifié)

### 2.2 `attendee-ems-brain` — 5 fichiers

- `workstreams/api-scaling-lfd2026/README.md` (nouveau)
- `backlog/BACKLOG-TECH.md` (modifié — entrées LFD)
- `NOW.md`, `INBOX.md`, `DEBTS.md` (modifiés)

---

## 3. TOUS les leviers (depuis mercredi soir)

> « Levier » = toute action visant à augmenter la capacité / fiabilité, ou à rendre le test
> possible. Statut de déploiement et propreté indiqués pour chacun.

| # | Levier | Où | Commit | Déployé | Propreté |
| --- | --- | --- | --- | --- | --- |
| L1 | **Stack staging isolée** (réplique iso-prod) | `docker-compose.staging.yml`, `.env.staging.example` | `c7247ae` | staging | ✅ propre |
| L2 | **pgBouncer** mode transaction + healthcheck | compose staging, `bd858f9` | `c7247ae`,`bd858f9` | staging | ✅ propre |
| L3 | **`directUrl` Prisma** (migrations sous pgBouncer) | `prisma/schema.prisma` | `d3f3421` | staging | ✅ propre (isolé) |
| L4 | **Dump/restore prod→staging** | `scripts/db-dump.sh` | `c7247ae` | n/a (outil) | ✅ propre |
| L5 | **Anonymisation PII (RGPD)** | `scripts/staging-anonymize.sql` | `1ba886f` | staging | ✅ propre |
| L6 | **Isolation projet Docker** `name=ems-staging` | compose | `5f72daa` | staging | ⚠️ correctif d'incident (voir §6) |
| L7 | **Email asynchrone (BullMQ)** | `email-queue.*`, `email.service.ts` | `d3f3421` | staging | ⚠️ noyé dans commit fourre-tout |
| L8 | **Worker séparable `PROCESS_ROLE`** (Voie A) | `process-role.ts`, `main.ts` | `d3f3421` | staging (défaut `all`) | ⚠️ noyé + reformatage |
| L9 | **Transaction d'inscription allégée** (COUNT capacité + anti-doublon sortis de la transaction, `P2002`→409) | `public.service.ts` | `d3f3421` | staging | ❌ diff illisible (voir §6) |
| L10 | **Pool DB / `connection_limit`** sizing | `.env.staging` | `c7247ae` | staging | ✅ propre |
| L11 | **Auth SMTP conditionnelle** (fix Mailpit) | `email.service.ts` | `47c82bf` | staging | ✅ propre |
| L12 | **`synchronous_commit=off`** | runtime Postgres | (hors git) | testé → **abandonné** (aucun gain, remis `on`) | ✅ documenté |
| L13 | **Cluster Node multi-worker** (Voie B) | runtime staging | (hors git) | testé staging | ❌ **interdit prod** sans workstream clustering |
| L14 | **Scripts k6** (baseline, stress, crash, endurance, spike) | `scripts/k6/*` | plusieurs | staging | ✅ propres |

**Récapitulatif d'état :**
- Déployés sur **staging uniquement** : L1–L11.
- Déployés en **prod** : **aucun**.
- **Testés puis abandonnés** : L12 (`synchronous_commit`).
- **Interdits en prod** tant que le workstream clustering n'est pas livré : L13 (Voie B).

---

## 4. Campagnes de test exécutées + résultats RÉELS

> Mesures réelles sur staging (générateur k6 **mutualisé** sur le VPS = plancher prudent).
> Production surveillée à chaque palier : `health=200`, CPU ≈ 0 % → **jamais impactée**.

| Campagne | Script | Charge | Résultat réel |
| --- | --- | --- | --- |
| **Crash global (mixte)** | `crash-test-full.js` | browse 0→150 + register 0→300 + dashboard 0→40 | register **705 abouties (2,9/s)**, 1 697 échecs ; browse p95 7,53 s err 9,86 % ; dashboard p95 60 s (plafond) err 59,27 % ; login p95 34,16 s err 0 % ; `http_req_failed` **31,57 %** |
| **Inscription isolée** | `register-stress.js` | paliers 50→400 VU | **8 448 abouties (25,4/s)**, erreurs métier 2,33 % ; `http_req_failed` 1,16 % ; reg p95 8,94 s / max 12,39 s |
| **Spike (absorption)** | `spike-absorption.js` | 3 vagues @400 VU, ramp 10 s | **6 237 abouties (~23,8/s servies)**, p95 9,38 s / max 12,01 s ; échec rafale 10,5 % ; **résorption immédiate** (CPU→0 % entre vagues) |
| **Endurance (fiabilité)** | `endurance-reliability.js` | 60 VU mixte, 10 min | **26 694 opérations, 0,00 % erreur** ; médian 0,32 s / p95 2,55 s ; mémoire stable (~1,0 Go DB, ~0,3 Go app) |

**Profil ressources (staging) :** API ≈ 1,2–2,8 cœurs, PostgreSQL ≈ 2 cœurs, pgBouncer < 15 %.
**Goulot réel : saturation CPU par inscription** sur boîte mutualisée (DB **non** saturée :
1–4 connexions actives / 100).

> ✅ Les rapports `2026-06-09-crash-test-annuel.md` et `2026-06-24-rapport-client-fiabilite-pics.md`
> ont été **remis aux vrais chiffres** ci-dessus (le 26/06).

---

## 5. Backlog ajouté

- **`attendee-ems-brain/backlog/BACKLOG-TECH.md`** — entrées LFD 2026 : BullMQ async, scaling pic
  inscription, Puppeteer/QR. (commit `ab99893`)
- **`attendee-ems-brain/workstreams/api-scaling-lfd2026/README.md`** — pilotage du workstream.
- **`attendee-ems-back/docs/workstreams/api-scaling-clustering/README.md`** — plan clustering
  (Voie B) en 6 phases, prérequis Redis-adapter + présence Redis + sticky.
- **`attendee-ems-back/docs/workstreams/async-badge-email/README.md`** — externalisation badge/email.
- MAJ `NOW.md`, `INBOX.md`, `DEBTS.md`.

---

## 6. Problèmes de propreté à corriger

### P1 — Commit fourre-tout `d3f3421 feat(scaling)`
Un seul commit mélange **4 leviers de nature différente** (L3, L7, L8, L9) + **3 rapports** +
**1 handoff** + un **reformatage automatique complet** (guillemets simples→doubles, re-wrapping)
sur `main.ts` et `public.service.ts`.

Conséquence sur `public.service.ts` : **517 ajouts / 381 suppressions** alors que le changement
métier réel (sortir le `COUNT` capacité et l'anti-doublon de la transaction, `catch P2002`→409)
ne fait qu'**une cinquantaine de lignes**. Le reste est du bruit de reformatage → **diff
irrevisable**, **revert granulaire impossible**, risque de régression caché.

### P2 — Incohérence de chiffres entre les docs
Les docs brain (`api-scaling-lfd2026/README.md`) affirment **« ~33 inscriptions/s »** et
**« plafond = sérialisation écriture DB (ni CPU, ni pool, ni nb de process — prouvé) »**.
Les **mesures réelles** donnent **~25/s** avec **plafond = saturation CPU** (DB sous-utilisée).
→ Conclusion et chiffre à **réconcilier** (la version mesurée fait foi).

### P3 — Incident prod 502 pendant la session
Le commit `5f72daa` corrige une **collision de nom de projet Docker** (`backend`) entre la stack
staging et la prod, qui a provoqué un **502 en prod**. À transformer en **post-mortem** (cause,
détection, correctif `name=ems-staging`, garde-fou).

### P4 — `docker-compose.prod.yml` modifié sur la branche
La branche contient des modifications du compose **prod** (+52). À **ne pas déployer** tant
qu'elles ne sont pas isolées et revues séparément.

---

## 7. Plan d'annulation + refonte propre

> Principe : **ne rien perdre** (la branche `staging` reste la référence/archive) mais **rejouer
> chaque levier proprement**, un par un, diff minimal, mesure avant/après, dans son workstream.

### Étape 0 — Geler & archiver
- Taguer l'état actuel : `git tag archive/staging-2026-06-25` (back + brain). Référence figée.
- Ne **jamais** déployer la branche telle quelle en prod (cf. P4).

### Étape 1 — Séparer reformatage et logique
- Isoler le bruit : appliquer Prettier **seul** dans un commit dédié `style: prettier` (ou
  configurer l'éditeur pour ne pas reformater à la sauvegarde), de sorte que les commits de
  logique soient **diff minimal**.

### Étape 2 — Rejouer chaque levier dans une branche + un commit propres

| Ordre | Levier | Branche cible | Workstream de rattachement |
| --- | --- | --- | --- |
| 1 | L9 transaction allégée (`public.service.ts`) — **diff minimal** + test de non-régression inscription | `feat/register-tx-slim` | `api-scaling-lfd2026` |
| 2 | L7 email async BullMQ | `feat/email-async-bullmq` | `async-badge-email` |
| 3 | L8 worker `PROCESS_ROLE` (Voie A) | `feat/process-role-worker` | `api-scaling-clustering` (prérequis) |
| 4 | L3 `directUrl` Prisma | `chore/prisma-directurl` | `api-scaling-lfd2026` |
| 5 | L1/L2/L10 stack staging + pgBouncer + pool | `infra/staging-stack` | `api-scaling-lfd2026` |

> Chaque branche : 1 levier, 1 PR, 1 mesure avant/après, 1 entrée workstream. Aucun mélange.

### Étape 3 — Réconcilier la doc (P2)
- Corriger `api-scaling-lfd2026/README.md` : remplacer « ~33/s / plafond DB » par les **mesures
  réelles** (~25/s, plafond CPU, DB sous-utilisée). Les rapports infra font déjà foi (corrigés).

### Étape 4 — Post-mortem incident prod (P3)
- Créer `attendee-ems-brain/bugs/2026-06-25-prod-502-collision-compose.md` (cause, timeline,
  correctif, garde-fou `name=ems-staging` + check de pré-déploiement).

### Étape 5 — Isoler le compose prod (P4)
- Sortir les modifications de `docker-compose.prod.yml` dans une PR dédiée, revue à part, **non
  fusionnée** tant que non validée.

### Étape 6 — Ce qui reste interdit / en attente
- **L13 cluster (Voie B)** : reste **interdit en prod** jusqu'à livraison du workstream
  `api-scaling-clustering` (Redis-adapter + présence Redis + sticky + tests cross-worker verts).
- **L12 `synchronous_commit`** : abandonné, ne pas rejouer (aucun gain mesuré).

---

## 8. Garde-fous (rappel)

- ❌ **Jamais** toucher la prod. Toute opération scoped `ems-staging`.
- Vérifier `curl -sk https://127.0.0.1/api/health` (= 200) après chaque manip.
- Ne **pas** faire de `git checkout` du répertoire partagé `/opt/ems-attendee/backend` (prod).
- **Aucun** chiffre falsifié dans les rapports : seules les mesures réelles font foi.
