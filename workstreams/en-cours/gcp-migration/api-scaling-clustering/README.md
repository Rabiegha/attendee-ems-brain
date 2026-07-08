# Workstream — API scaling : clustering multi-worker

> **Statut :** EN COURS — Phase 1 (audit) faite ✅, exécution à démarrer.
> **Risque :** 🔴 Moyen/élevé (touche le printing temps réel via WebSocket)
> **Repo principal :** `attendee-ems-back`
> **Effort estimé :** ~1,5–2,5 semaines de dev effectif (détail §6).
> **Démarrage :** **juste après le chantier A (refonte)** — voir séquencement §8.
> **Créé :** 2026-06-24 · **Déplacé dans le brain :** 2026-07-06
> **Lié à :** [async-architecture](../../a-faire/async-architecture/README.md), [LFD 2026 load test](../../../infra/lfd-2026-load-test-plan.md), [ADR-005 print-queue-authority](../../../../attendee-context-hub/context/decisions/adr-005-print-queue-authority.md)

> 🔗 **Prérequis bloquant du levier L13 (Voie B / cluster).** Tant que ce workstream
> n'est pas livré, la Voie B est **interdite en prod** — voir
> [infra-scaling-pca](../infra-scaling-pca/README.md) et le cadre interdit du
> [chantier A](../lfd2026/00-plan-action.md).

---

## 1. Pourquoi ce n'est PAS un quick win

Passer l'API NestJS de 1 process à N workers (cluster Node / replicas Docker) augmente la
capacité check-in, mais **casse silencieusement le printing temps réel** si fait naïvement.

Le système d'impression dépend de deux registres **en mémoire de process** :

| Registre                            | Emplacement                                                                                     | Rôle                                                                 |
| ----------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `EventsGateway.emsClients`          | [websocket.gateway.ts](../../../../attendee-ems-back/src/websocket/websocket.gateway.ts)        | Présence des EMS Print Clients connectés (socketId, orgId, deviceId) |
| `PrintQueueService.exposedPrinters` | [print-queue.service.ts](../../../../attendee-ems-back/src/print-queue/print-queue.service.ts)  | Imprimantes exposées par chaque Print Client                         |
| Rooms Socket.IO `org:<id>`          | adapter in-memory par défaut                                                                    | Routage des broadcasts                                               |

En multi-worker, chaque worker a **sa propre mémoire**. Le Print Client est connecté en
WebSocket au Worker B ; une requête REST `POST /print-queue/add` (dashboard/mobile) peut
arriver sur le Worker A. Le Worker A **ne connaît ni le socket ni les imprimantes** du client.

```
Print Client Electron ──WS──> Worker B   (connaît socket + imprimantes)
Dashboard / Mobile ──POST──> Worker A     (ne connaît rien → job non livré)
```

### Les sticky sessions ne suffisent pas

Les sticky sessions épinglent **un client donné** à un worker. Mais le Print Client, le
dashboard et le mobile sont **des clients différents** → ils peuvent atterrir sur des workers
différents. Sticky = nécessaire (stabiliser les WS longues), **insuffisant seul**.

### Pire qu'une émission perdue : corruption d'état métier (ADR-005)

`addPrintJob` décide PENDING vs OFFLINE via `isEmsClientConnected()`, qui lit la **Map locale** :

```ts
const isClientOnline = this.eventsGateway.isEmsClientConnected(callerOrgId);
status = isClientOnline ? "PENDING" : "OFFLINE";
```

Sur le Worker A, un client connecté au Worker B est **invisible** → le job est marqué
**OFFLINE alors que le client est en ligne** → **l'invariant d'ADR-005 est violé**.

> ⚠️ Le `@socket.io/redis-adapter` règle l'**émission** inter-workers, mais **PAS** les
> requêtes de présence (`isEmsClientConnected`, `getConnectedDeviceIds`). La présence doit
> elle aussi être externalisée en Redis.

---

## 2. Audit Phase 1 — état en mémoire vs déjà persisté

Vérifié dans le code (2026-06-24).

### À externaliser (en mémoire, perdu au restart / non partagé)

- `EventsGateway.emsClients` — présence des Print Clients.
- `PrintQueueService.exposedPrinters` — registre des imprimantes.
- Rooms Socket.IO — via redis-adapter.

### Déjà sûr (persisté en DB — NE PAS re-migrer)

- Print jobs `PENDING` / `OFFLINE` / `FAILED` / `COMPLETED` → table `printJob` (Prisma).
  `getPendingJobs`, `getOfflineJobs`, `retryOfflineJobs`, `dismissOfflineJobs` lisent/écrivent
  la DB. **L'état des jobs survit déjà à un redémarrage de worker.**

> Conséquence : le chantier porte sur **2 registres de présence + l'adapter**, pas sur la
> file de jobs. Périmètre plus petit donc plus sûr que l'audit initial ne le laissait penser.

---

## 3. Plan de migration progressif

| Phase                           | Objectif                                                                                                                                                                                 | Sortie de phase                    |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| **1 — Audit**                   | Cartographier tout l'état en mémoire WS/printing (✅ fait ci-dessus)                                                                                                                     | Liste figée                        |
| **2 — Registres Redis**         | `PrinterRegistryService` + **présence (`emsClients`)** déplacés en Redis (`online/offline`, `lastSeenAt`, `clientId`, `orgId`, `printers`). **Garder 1 worker.** Comportement identique. | Iso-fonctionnel en mono-worker     |
| **3 — Socket.IO redis-adapter** | Ajouter `@socket.io/redis-adapter` (IoAdapter NestJS custom). Émissions inter-workers OK en staging.                                                                                     | Broadcast cross-worker validé      |
| **4 — Sticky sessions nginx**   | `ip_hash`/sticky + upgrade WebSocket. Stabilité des connexions longues.                                                                                                                  | WS stables sous reload             |
| **5 — Staging multi-worker**    | **2 workers en staging uniquement.** Tests printing cross-worker + check-in + reconnect.                                                                                                 | Suite de non-régression verte      |
| **6 — Prod progressive**        | 1 worker si event proche → 2 workers après validation → 4 workers après observation logs/métriques.                                                                                      | Rollback documenté à chaque palier |

> **Ordre critique :** la présence (`emsClients`) doit migrer en Redis **en même temps** que
> `exposedPrinters` (Phase 2). Sinon ADR-005 reste cassé même après l'adapter (Phase 3).

---

## 4. Tests de non-régression obligatoires (Phase 5)

- [ ] Print Client connecté au Worker B, requête print envoyée au Worker A → job livré.
- [ ] Job correctement marqué PENDING (client online) même quand la requête frappe un autre worker.
- [ ] Déconnexion / reconnexion du Print Client → présence Redis mise à jour, OFFLINE→PENDING au retour.
- [ ] Plusieurs Print Clients dans la même org.
- [ ] Plusieurs imprimantes exposées (dédup `deviceId::printerName`).
- [ ] Redémarrage d'un worker pendant un event → aucun job perdu, registres reconstruits depuis Redis.
- [ ] Check-in massif + print simultanés → pas de régression de latence.

---

## 5. Prérequis AVANT d'activer le multi-worker en prod

1. `@socket.io/redis-adapter` installé + IoAdapter custom branché.
2. Sticky sessions nginx + WebSocket upgrade.
3. `exposedPrinters` **et** `emsClients` sortis en Redis (source de vérité partagée).
4. Suite de tests cross-worker verte en staging.

Tant que ces 4 points ne sont pas faits : **rester en 1 worker**. Les autres leviers de
capacité (pgBouncer, pool DB, async badge/email via BullMQ) se font **sans** clustering et
sont à prioriser pour LFD 2026.

---

## 6. Estimation d'effort

Dev solo, tests inclus (pas juste le chemin heureux).

| Phase | Contenu | Estimation |
| --- | --- | --- |
| **2 — Registres Redis** | Sortir `emsClients` **et** `exposedPrinters` en Redis, garder 1 worker, iso-fonctionnel. C'est le cœur du risque (ADR-005). | **2–4 j** |
| **3 — redis-adapter** | `@socket.io/redis-adapter` + IoAdapter NestJS custom, broadcasts cross-worker validés en staging. | **1–2 j** |
| **4 — Sticky nginx** | `ip_hash`/sticky + upgrade WebSocket, stabilité des WS longues. | **0,5–1 j** |
| **5 — Staging multi-worker** | 2 workers + la suite de non-régression §4 (7 scénarios). C'est là que les vrais bugs sortent. | **2–4 j** |
| **6 — Prod progressive** | 1→2→4 workers avec observation entre paliers + rollback documenté. Étalé dans le temps. | **1–2 j actifs** |

**Total : ~1,5 à 2,5 semaines de travail effectif**, réparti sur ~3–4 semaines calendaires
(phase 6 volontairement lente).

**Ce qui peut faire déraper :**

- **Présence en Redis (phase 2)** = le vrai piège : un edge case ADR-005 qui passe (job
  OFFLINE alors que client online sur un autre worker) est **silencieux** → debug pénible.
  C'est là que je mets la marge.
- **Reconstruction des registres après restart worker** (test phase 5) — souvent sous-estimé.
- **Reproduire un staging fidèle à la prod** (nginx, WS upgrade) avant de tester.

**Ce qui le rend plus court que l'audit initial :** les print jobs sont **déjà en DB** → on
ne migre que 2 registres de présence + l'adapter, pas la file de jobs.

---

## 7. Vision scaling horizontal — ce workstream est la FONDATION

Ce chantier (état → Redis, mono-serveur multi-worker) est le **palier 0** d'une ambition plus
large : répartir la charge sur plusieurs serveurs / clouds. Voici la cible et ses limites.

### 7.1 Deux objectifs à ne pas confondre

- **Capacité** (encaisser le pic, 3000 inscriptions simultanées) → **scaling horizontal de l'API**.
- **Résilience** (« être tranquille », survivre à la panne d'un DC) → **redondance multi-site**.

Le **multi-cloud OVH + GCP** répond surtout à la **résilience**, presque pas à la capacité.

### 7.2 Le mur nº1 : l'état en mémoire (ce workstream)

Ajouter des serveurs **sans** externaliser l'état (présence + imprimantes) casse l'impression
temps réel **entre datacenters** au lieu d'entre workers. Redis-adapter + présence Redis = le
préalable **non négociable** de tout scaling horizontal, mono comme multi-serveur.

### 7.3 Le mur nº2 : Postgres n'a qu'UN primaire (writer)

Les inscriptions = des **écritures**. Quel que soit le nombre de serveurs d'API, ils écrivent
dans **la même** DB primaire. → le plafond d'écriture ne bouge pas en ajoutant du compute.

### 7.4 Paliers recommandés

| Palier | Quoi | Gagne quoi | Effort |
| --- | --- | --- | --- |
| **0. Fondation (CE workstream)** | État → Redis, adapter, sticky. 1 serveur, N workers. | Capacité pour le pic sans casser l'impression. | ~1,5–2,5 sem |
| **1. Multi-conteneurs 1 host** | LB local + Docker (Compose `--scale` → **Swarm** si besoin), même Redis/PG. | Scale la capacité, simple à opérer. | qq jours |
| **2. HA actif-passif** | 2ᵉ serveur en **standby** : réplica Postgres streaming + bascule DNS/LB. | La « tranquillité » : survit à la perte d'un DC. | 1–2 sem |
| **3. Actif-actif multi-région** | Trafic servi des 2 côtés, DB multi-master. | Peu de gain de capacité, énorme complexité. | ❌ overkill |

**Reco : viser 0 → 1 → 2.** Le palier 2 (actif-**passif**) donne ~95 % de la résilience
recherchée pour une fraction du coût de l'actif-actif.

### 7.5 Pourquoi le multi-région actif-actif gagne peu (contre-intuitif)

« 2 serveurs → 2× de compute » est **vrai** — mais on l'obtient avec **2 serveurs dans la même
région**, à côté de la DB. En multi-cloud, la DB est chez **un seul** des deux :

```
Serveur OVH  ──(écrit local, ~0.5ms)─►  Postgres (OVH)
Serveur GCP  ──(écrit distant, +10–30ms/requête)─►  Postgres (OVH)
```

Une inscription fait plusieurs allers-retours DB → le serveur « loin » devient **plus lent par
requête** et contribue **moins** (~1,3–1,5× au lieu de 2×), pour le **même plafond DB**, plus le
risque de réplication/split-brain. → Garder le 2ᵉ cloud pour la **résilience (actif-passif)**,
pas pour la capacité.

### 7.6 Règle de décision : CPU ou DB ? (à mesurer AVANT d'acheter un 2ᵉ serveur)

- Plafond **CPU** (compute) → ajouter des serveurs monte le débit fort. ✅
- Plafond **DB** (écritures) → 2 serveurs ne changent presque rien ; alléger la DB d'abord
  (transaction slim, pgBouncer, pool).

⚠️ **Point non résolu :** la doc dit « ~33/s plafond DB », les mesures disent « ~25/s plafond
CPU ». À **trancher à l'étape 3 du chantier A** — c'est CE chiffre qui décide si un 2ᵉ serveur
apporte 2× ou ~0.

### 7.7 Contraintes de contexte

- **Souveraineté (contrat MEAE = gouvernement français).** Mettre des données participants sur
  **GCP** peut poser un problème réglementaire, même en région Paris. À vérifier au CDC **avant**
  d'architecturer autour de GCP. OVH (souverain FR) est plus sûr côté conformité. Voir
  [appel migration GCP](../../../infra/lfd-2026-migration-gcp-call.md).
- **Orchestration Docker :** **Compose** (1 host) → **Swarm** (multi-host léger) suffisent en
  solo. **Kubernetes** = puissant mais lourd à opérer seul → à éviter sans vrai besoin permanent.

---

## 8. Séquencement & dépendances

- **Démarre juste après le chantier A (refonte)** : la refonte rejoue proprement les leviers
  non-cluster (pgBouncer, pool DB, async email BullMQ, tx slim) qui donnent déjà la capacité
  d'inscription **sans** clustering. On attaque le cluster sur une base saine et mesurée.
  Voir [chantier A](../lfd2026/00-plan-action.md) et le [NOW](../../../workspace-rabie/NOW.md).
- **Prérequis d'entrée :** étape 3 du chantier A tranchée (CPU vs DB, §7.6) → on sait si le
  clustering apporte vraiment quelque chose avant d'investir 1,5–2,5 sem.
- **Repoussable :** ce workstream n'est **pas** sur le chemin critique du contrat MEAE. Si
  l'échéance (4–5 sept 2026) serre, il reste derrière le chantier A sans risque contractuel.
