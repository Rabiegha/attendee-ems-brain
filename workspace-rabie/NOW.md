# NOW — Focus actuel

## Focus actif

```
Workstream: Refonte propre LFD 2026 (chantier A) — "tout rejouer sans rien perdre"
App:        attendee-ems-back
Status:     Active
Priority:   High (contrat MEAE, 4–5 sept 2026)
```

**Objectif :** Repartir d'une base propre depuis `main` et **rejouer chaque levier de scaling**
dans son propre commit (diff minimal + mesure avant/après). La branche `staging` reste
l'**archive de référence** — on ne perd rien.

**Plan maître :** [00-plan-action.md](../workstreams/en-cours/lfd2026/00-plan-action.md) (chantier A, §0–§2)
· **Workstream :** [infra-scaling-pca](../workstreams/en-cours/infra-scaling-pca/README.md)
· **📌 Suivi vivant (état par levier) :** [01-suivi-leviers.md](../workstreams/en-cours/lfd2026/A-I-leviers/01-suivi-leviers.md) — **à ouvrir au début de chaque chat**.

**Prochaine action :**
➡️ **Focus du jour : L9 puis L9.1**, **codés ET testés** (non-régression). Merger d'abord la MR [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) (L1/L2/L10) sur `dev`.

### 📌 Priorités demain (15/07)

- Warm-up Mailgun dès l'arrivée + petite montée en charge.
- Retry souscription plan Mailgun + prise GitHub Pro si besoin.
- Protéger `main` et `staging` avec checks requis.
- Demander/clarifier les certificats iOS + Android.
- Valider les modifications déjà livrées sur T et C2.
- Finaliser 0-MON côté VPS (runbook exécutable + checklist).
- Prépa call GCP jeudi 11h30 (infra actuelle, archi actuelle, questions event).
- Fin de journée: MAJ suivi + journal + rapport manager Cem.

➡️ Détail exécutable : [journal/2026-07-15-todo.md](./journal/2026-07-15-todo.md)

### 🎯 Ordre de priorité (à jour 2026-07-08)

1. **L9** — transaction d'inscription allégée (`feat/register-tx-slim`) **+ test de non-régression**.
2. **L9.1** — compteur présence session O(1) (`feat/session-present-counter`, colonne PG par défaut) **+ test**.
3. **Chantier Cloud Run — génération PDF billet** (chantier B) : billet PDF joint à l'email (Gotenberg / Cloud Run).
4. **Prépa du call de demain** : **post-event** + **review du setup Cloud Run** déjà effectué — points clés : [tenir l'event **sans GCP** (+ gate post-L9)](../workstreams/en-cours/lfd2026/K-resilience-event/tenir-event-sans-gcp.md) · [workstream migration GCP (archi cible B + questions Entiovi)](../workstreams/a-faire/gcp-migration/README.md).
5. **Ensuite seulement** : reste des leviers/étapes du chantier A, workstream capacité live forte charge, **backlog mobile** (cf. « Ensuite »).

> ⚠️ **Validation post-leviers OBLIGATOIRE** : une fois L9 **et** L9.1 appliqués, **re-tester que tout
> marche** (inscription + check-in), **surtout L9.1** — le compteur ne doit **jamais diverger du réel**
> (pas de survente, occupation correcte après IN / OUT / annulation, idempotence double-scan).

### Étapes du chantier A (~3 h 35, hors temps k6)

> État détaillé par levier/branche → [01-suivi-leviers.md](../workstreams/en-cours/lfd2026/A-I-leviers/01-suivi-leviers.md).

- [x] **0.** Geler & archiver (tag `archive/staging-2026-06-25` + branche `staging-archive-2026-06-25`).
- [x] **1.** Séparer reformatage / logique (format-on-save vérifié OFF).
- [ ] **2.** Rejouer les **leviers du chantier A (+ L9.1)**, 1 branche + 1 commit + 1 mesure chacun — **1/6 fait** :
  - 🟢 L1/L2/L10 stack staging + pgBouncer + pool — `infra/staging-stack` (codé+testé local, MR #14)
  - ⚪ L9 transaction allégée — `feat/register-tx-slim` **(focus du jour)**
  - ⚪ L9.1 compteur présence session O(1) — `feat/session-present-counter` **(focus du jour)**
  - ⚪ L7 email async BullMQ — `feat/email-async-bullmq`
  - ⚪ L8 worker `PROCESS_ROLE` (Voie A) — `feat/process-role-worker`
  - ⚪ L3 `directUrl` Prisma — `chore/prisma-directurl`
- [ ] **3.** Réconcilier la doc : trancher le chiffre réel (**~25/s plafond CPU** mesuré vs « ~33/s DB » écrit) dans `infra-scaling-pca/README.md`.
- [x] **4.** Post-mortem 502 prod → [bugs/fait/2026-06-25-prod-502-collision-compose.md](../bugs/fait/2026-06-25-prod-502-collision-compose.md) (garde-fous → [levier A-I](../workstreams/en-cours/lfd2026/A-I-leviers/garde-fous-deploiement-staging.md)).
- [ ] **5.** Isoler le compose prod dans une **PR dédiée** (ne pas merger sans revue).
- [ ] **6.** **Cadrer l'interdit** (note, pas de code) — voir bloc ci-dessous.

### 🚫 Cadre interdit / en attente (étape 6)

- **L13 cluster (Voie B) = INTERDIT en prod** tant que le workstream
  [api-scaling-clustering](../workstreams/en-cours/api-scaling-clustering/README.md)
  n'est pas livré (Redis-adapter + présence Redis + sticky nginx + tests cross-worker verts).
  Sinon → casse l'impression temps réel **silencieusement** (registres en mémoire de process).
  ℹ️ Code du plan clustering : `attendee-ems-back` (branche `staging`,
  `docs/workstreams/api-scaling-clustering/`) — **pas encore sur `main`**.
- **L12 `synchronous_commit`** : abandonné, **ne pas rejouer** (aucun gain mesuré).

---

## ⏭️ Ensuite — dans l'ordre (à jour 2026-07-08)

1. **L9 + L9.1** (focus du jour) — codés + **testés** (non-régression), cf. ci-dessus.
2. **Chantier B — chaîne email → billet PDF** : joindre le **billet PDF** à l'email (moteur **Gotenberg sur Cloud Run**). Une partie à décider.
   ➡️ [00-plan-action.md §3-B](../workstreams/en-cours/lfd2026/00-plan-action.md) · [décision lib PDF/badge](../workstreams/en-cours/lfd2026/B-email-billet-pdf/lib-pdf-badge.md).
3. **Prépa du call de demain** : **post-event** + **review du setup Cloud Run** déjà effectué.
4. **Reste du chantier A** : merger #14 sur `dev`, puis L7 / L8 / L3, puis étapes 3→6 (réconcilier CPU/DB, post-mortem 502, isoler compose prod, cadrer interdits).
5. **Workstream capacité live forte charge** (sessions/inscriptions LFD) — portier Redis + WebSocket + pic combiné.
   ➡️ [02-capacite-live-forte-charge](../workstreams/a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md).
6. **Backlog mobile LFD2026** : pagination sessions, offline scans fiable, perf liste ~20k attendees, refactor check-in/out historique, bug recherche entreprise…
   ➡️ [appli-mobile-lfd2026](../backlog/en-cours/appli-mobile-lfd2026.md).
7. **API scaling — clustering multi-worker** (post-event) : présence + imprimantes en **Redis**, `redis-adapter`, sticky nginx, tests cross-worker. Fondation du scaling horizontal — **le verrou de la migration GCP**.
   ➡️ [workstream api-scaling-clustering](../workstreams/en-cours/api-scaling-clustering/README.md).
8. **Migration GCP complète** (post-event) — **archi cible Stratégie B** (Cloud SQL HA/PITR, Memorystore, LB, MIG, Cloud Run), un seul cutover, mono-instance au début.
   ➡️ [workstream gcp-migration](../workstreams/a-faire/gcp-migration/README.md).

---

## ⏸️ En pause — Stabilisation mobile

Éjections (hard restart + ouverture scan). Reste : valider sur tablette physique les fixes
socket/hors-ligne du 18/06.
➡️ [workstreams/fait/mobile-eject-socket-resilient-delta-async/01-eject-hard-restart.md](../workstreams/fait/mobile-eject-socket-resilient-delta-async/01-eject-hard-restart.md).

---

## Fait récemment (2026-06-18)

- ✅ **Socket résilient + delta-sync à la reconnexion** livré ([workstreams/mobile-stabilization/03](../workstreams/fait/mobile-eject-socket-resilient-delta-async/03-socket-resilient-delta-sync.md) §8) : reconnexions infinies, delta-sync auto, indicateur d'état socket.
- ✅ **Détection hors-ligne rapide** (~4-6s au lieu de ~30-48s) + **retry silencieux 409 STALE**.
- ✅ Bugs corrigés : crash écran gris (Rules of Hooks dans `ConflictsBanner`), faux conflit STALE sur action + action inverse offline, build EAS débloqué (runtimeVersion).
- ⏳ Reste : valider sur tablette physique (APK preview EAS) les scénarios de coupure réseau.

---

## Objectifs de cette semaine

- [ ] **L9** (transaction allégée) codé + **testé** (non-régression) — `feat/register-tx-slim`.
- [ ] **L9.1** (compteur présence session O(1)) codé + **testé** — `feat/session-present-counter`.
- [ ] **Validation post-leviers** : vérifier que tout marche après L9 **et** L9.1, **surtout L9.1** (pas de divergence du compteur).
- [ ] **Chantier Cloud Run** (email → billet PDF) : décider + attaquer la 1re partie.
- [ ] **Prépa call de demain** : post-event + review du setup Cloud Run.
- [ ] Merger la MR [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) (L1/L2/L10) sur `dev`.

---

## Hors-scope maintenant

- **Cluster Voie B en prod** (L13) — interdit tant que `api-scaling-clustering` non livré.
- **Migration GCP big-bang** — reportée (chantier F).
- Wallet Apple/Google (chantier G — V2).
- Refonte navigation mobile / migration persistance (MMKV) — mobile en pause.
- Async Architecture (back) — pas le focus ce cycle.

---

## Règle de mise à jour

- Ne pas recréer ce fichier. Le mettre à jour **quand le focus change**.
- Idées non prioritaires → [INBOX.md](INBOX.md).
