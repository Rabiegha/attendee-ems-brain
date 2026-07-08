# LFD 2026 — Stratégie : 2 chantiers (Prépa event / Post-event)

> **Objet :** cadrage stratégique issu des décisions du 8 juillet 2026. Sépare clairement ce qui est
> fait **pour l'event** (4-5 sept 2026, ~12 400 inscriptions, **3000 simultanés** au pic) de ce qui
> est un **investissement post-event** (scaling horizontal + migration complète OVH → GCP).
> **Rôle :** carte de haut niveau — point d'entrée pour « enchaîner » sur la suite.
> **Date :** 2026-07-08.

---

## TL;DR — les décisions qui structurent tout

1. **L'event se joue sur OVH.** Scaling **vertical** (plus gros VPS) + **leviers de code** + **offload
   Gotenberg**. Pas de migration, pas de clustering avant le 4 sept (trop risqué à 5 semaines).
2. **GCP pour l'event ≈ Cloud Run** (rendu badges/Gotenberg, hors VPS) + éventuel **filet DR**. Le
   reste de GCP (Cloud SQL, Memorystore, LB, migration) = **valeur post-event**.
3. **Co-localiser app + DB.** Jamais app OVH + DB Cloud SQL cross-cloud (latence inter-cloud tue le
   débit écriture). Un seul chemin chaud.
4. **Le scaling horizontal est bloqué par le CODE, pas par le cloud.** L'app garde de l'état en RAM
   (imprimantes, présence, socket.io) → il faut la rendre **stateless** (état en Redis) **avant** de
   multiplier les instances. Ce refacto ne peut pas être délégué à l'infra GCP.
5. **Mesurer AVANT de sur-investir.** Baseline k6 + mesure par levier. Les leviers + l'offload
   Gotenberg peuvent lever le plafond (~25/s CPU) assez pour **ne pas avoir besoin** du clustering
   pour l'event.
6. **Une file d'attente absorbe les pics.** ~33/s **soutenu** peut servir un pic instantané bien plus
   haut (le tampon lisse la rafale).

---

## Chantier 1 — Prépa event (échéance 4-5 sept 2026)

> Objectif : tenir **3000 inscriptions quasi simultanées** + affichage temps réel, **sur OVH**, sans
> survente et sans saturer Postgres.

| Volet | Contenu | Où |
|---|---|---|
| **A — Refonte propre (leviers)** | L9 (tx courte) · L7 (email async) · L8 (worker) · L3 (directUrl) — 1 levier = 1 branche = 1 commit + mesure | [00-plan-action.md](../workstreams/en-cours/lfd2026/00-plan-action.md) · [suivi leviers](../workstreams/en-cours/lfd2026/01-suivi-leviers.md) |
| **Mesure** | Baseline k6 + mesure par levier (réconcilier ~25/s CPU vs ~33/s DB) | [load-test-plan](lfd-2026-load-test-plan.md) |
| **Offload PDF** | Gotenberg sur **Cloud Run** — sort le rendu badge (gros CPU) du VPS | plan-action §3-B |
| **Capacité live forte charge** | Redis **portier atomique** (anti-survente) + **file BullMQ** (tampon) + **WebSocket mono-instance** (live) + **pré-chauffe cache** (anti-stampede) | [workstream 02](../workstreams/a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md) |
| **Inscriptions par session** | Lien public/session + capacité/waitlist + refonte front | [sessions-inscriptions-lfd2026](../workstreams/a-faire/sessions-inscriptions-lfd2026/README.md) |
| **Fondations minimales** | CI/CD minimal + monitoring/alerting minimal | plan-action §P0 |
| **Autres** | ESP warm-up (J1) · sauvegarde DB auto · sécurité QR | plan-action §3-C/E/D |

**Archi cible event (mono-instance) :**
- **Lecture** (statut, 3000 users) → **Redis / cache / WebSocket** (jamais Postgres).
- **Écriture** (inscription) → **portier `DECR` Redis** → **file BullMQ** → Postgres à son rythme.
- **Live** → WebSocket **emit direct** (mono-instance, pas de pub/sub).
- **DB** → OVH, co-localisée. Scaling **vertical** (IOPS/NVMe + tuning) si le débit soutenu l'exige.

**Rôle de GCP pour l'event :**
- ✅ **Cloud Run** (Gotenberg) — le seul usage « event » évident.
- 🟡 **DR / warm standby** (optionnel) — filet si OVH tombe le jour J.
- 🔴 Cloud SQL / Memorystore / migration → **post-event**.

---

## Chantier 2 — Post-event (le vrai scaling, au calme)

> Objectif : passer au **scaling horizontal** managé et migrer proprement OVH → GCP, **sans la
> pression de l'event**.

### 2.1 — Clustering / scaling horizontal

Rendre l'app **stateless** (prérequis du multi-instance) :

- **socket.io redis-adapter** (le WS écoute Redis pub/sub → toutes les instances synchronisées).
- **présence / imprimantes en Redis** (sorties de la RAM des process).
- **sticky sessions** côté nginx.
- **gate dur : pas de cluster en prod tant que les tests cross-worker ne sont pas verts** (sinon casse
  l'impression temps réel **silencieusement**).

➡️ Workstream : [api-scaling-clustering](../workstreams/en-cours/api-scaling-clustering/README.md).
Effort estimé ~1,5-2,5 sem. **C'est du code applicatif** (pas délégable à l'infra GCP).

### 2.2 — Migration complète OVH → GCP

- **Cloud SQL** (Postgres managé : HA + PITR + pooling) — co-localisé avec l'app.
- **Memorystore** (Redis managé, dont le pub/sub du redis-adapter).
- **Cloud Run / GCE** + **LB managé** + autoscaling.
- **DMS / réplication logique** pour la bascule à faible downtime.

➡️ Questions & critères de décision : [migration-gcp-call](lfd-2026-migration-gcp-call.md).
Continuité/HA (chantier F du plan) est **reporté ici** (Cloud SQL fournit HA + PITR natifs).

---

## Pourquoi cette séparation (le raisonnement)

- **Big-bang GCP-primary à 5 semaines de l'event = risque n°1.** Inverser (OVH primaire, GCP DR) est
  plus prudent → l'event sur une base connue et stabilisée.
- **Le clustering rushé en solo avant l'event = mode d'échec silencieux** (badges qui ne sortent plus).
  Le *plan* + un *POC* + un *spec pour l'équipe infra* sont faisables ; un cluster **prod-ready testé**
  ne l'est pas dans le temps imparti.
- **Mesurer d'abord** peut rendre le clustering **inutile pour l'event** : leviers + offload Gotenberg
  + scaling vertical suffisent probablement pour 3000 simultanés (surtout avec file d'attente + portier
  Redis qui protègent Postgres).

---

## Références (tout est ici)

**Learnings (concepts durables) :**
- [Scaling horizontal : stateless d'abord](../learnings/2026-07-08-scaling-horizontal-stateless.md)
- [Pic d'inscription temps réel (Redis portier + pull/push + cache)](../learnings/2026-07-08-pic-inscription-temps-reel.md)
- [pgBouncer vs pool](../learnings/2026-07-07-pgbouncer-et-pool-db.md) · [directUrl Prisma](../learnings/2026-07-07-directurl-prisma.md)

**Workstreams :**
- [lfd2026 — chaîne de livraison](../workstreams/en-cours/lfd2026/README.md) (plan maître : [00-plan-action](../workstreams/en-cours/lfd2026/00-plan-action.md))
- [Capacité live forte charge (02)](../workstreams/a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md)
- [infra-scaling-pca](../workstreams/en-cours/infra-scaling-pca/README.md) · [api-scaling-clustering](../workstreams/en-cours/api-scaling-clustering/README.md)

**Infra / décision :**
- [migration-gcp-call](lfd-2026-migration-gcp-call.md) · [capacity-planning](lfd-2026-capacity-planning.md) · [executive-summary](lfd-2026-executive-summary.md)
