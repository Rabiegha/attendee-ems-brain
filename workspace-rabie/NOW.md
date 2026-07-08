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
· **📌 Suivi vivant (état par levier) :** [01-suivi-leviers.md](../workstreams/en-cours/lfd2026/01-suivi-leviers.md) — **à ouvrir au début de chaque chat**.

**Prochaine action :**
➡️ **Levier suivant : L9** (transaction allégée, branche `feat/register-tx-slim`) — + merger la MR [#14](https://github.com/Rabiegha/attendee-ems-back/pull/14) (L1/L2/L10) sur `dev`.

### Étapes du chantier A (~3 h 35, hors temps k6)

> État détaillé par levier/branche → [01-suivi-leviers.md](../workstreams/en-cours/lfd2026/01-suivi-leviers.md).

- [x] **0.** Geler & archiver (tag `archive/staging-2026-06-25` + branche `staging-archive-2026-06-25`).
- [x] **1.** Séparer reformatage / logique (format-on-save vérifié OFF).
- [ ] **2.** Rejouer les **5 leviers**, 1 branche + 1 commit + 1 mesure chacun — **1/5 fait** :
  - 🟢 L1/L2/L10 stack staging + pgBouncer + pool — `infra/staging-stack` (codé+testé local, MR #14)
  - ⚪ L9 transaction allégée — `feat/register-tx-slim`
  - ⚪ L7 email async BullMQ — `feat/email-async-bullmq`
  - ⚪ L8 worker `PROCESS_ROLE` (Voie A) — `feat/process-role-worker`
  - ⚪ L3 `directUrl` Prisma — `chore/prisma-directurl`
- [ ] **3.** Réconcilier la doc : trancher le chiffre réel (**~25/s plafond CPU** mesuré vs « ~33/s DB » écrit) dans `infra-scaling-pca/README.md`.
- [ ] **4.** Post-mortem 502 prod → `bugs/2026-06-25-prod-502-collision-compose.md`.
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

## ⏭️ Ensuite — dans l'ordre

1. **Chantier A — refonte** (archivage + rejouer les leviers proprement) — étapes 0→6 ci-dessus.
2. **API scaling — clustering multi-worker** (juste après A) : externaliser présence +
   imprimantes en **Redis**, `redis-adapter`, sticky nginx, tests cross-worker. **Fondation de
   tout le scaling horizontal.** Effort ~1,5–2,5 sem. Prérequis : l'étape 3 de A (trancher
   CPU vs DB) doit dire si le cluster apporte vraiment quelque chose.
   ➡️ [workstream api-scaling-clustering](../workstreams/en-cours/api-scaling-clustering/README.md).
3. **Chantier B — chaîne email → billet PDF** : joindre le **billet PDF à l'email** de
   confirmation (aujourd'hui non joint). Moteur tranché : **Option D — Gotenberg sur Cloud Run**.
   **Une seule partie**, à décider laquelle.
   ➡️ [00-plan-action.md §3-B](../workstreams/en-cours/lfd2026/00-plan-action.md) ·
   [décision lib PDF/badge](../workstreams/en-cours/lfd2026/decisions/lib-pdf-badge.md).

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

- [ ] Dérouler le **chantier A (refonte)** — étapes 0 → 6 (checklist ci-dessus).
- [ ] Rejouer les 5 leviers proprement, 1 branche + 1 mesure chacun.
- [ ] Réconcilier le chiffre de plafond (**~25/s CPU** vs « ~33/s DB ») dans la doc.
- [ ] **Décider quelle partie** du chantier B (email → billet PDF) on attaque en premier.
- [ ] (Si temps) valider sur tablette physique les fixes mobile du 18/06.

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
