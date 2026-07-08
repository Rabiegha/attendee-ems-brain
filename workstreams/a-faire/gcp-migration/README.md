# Workstream — Migration GCP (post-event)

> **Objet :** faire passer la prod d'**OVH (VPS)** vers **GCP**, avec pour **archi cible la Stratégie B**
> (autoscaling MIG + Cloud SQL managé + Memorystore + LB), en **un seul cutover**. Ce workstream cadre
> la **proposition Entiovi**, l'**archi cible**, le **rôle d'OVH après migration**, et les **questions**
> à poser à l'agence.
> **Date :** 8 juillet 2026 · **Statut :** 🔵 à faire (**post-event**, aucun rush) · **Partenaire :** Entiovi
> **Réfs :** [analyse proposition Entiovi](./00-analyse-proposition-entiovi.md) ·
> [archi cible](./01-archi-cible.md) · [questions Entiovi](./02-questions-entiovi.md) ·
> [décision — tenir l'event sans GCP](../../en-cours/lfd2026/decisions/tenir-event-sans-gcp.md) ·
> [chantier clustering (le verrou)](../../en-cours/api-scaling-clustering/README.md) ·
> [00-plan-action §3-F](../../en-cours/lfd2026/00-plan-action.md)
> **Journal privé (hors git, credentials) :** `local-files/migration-private/MIGRATION_LOG.md`

---

## 0. Cadre & principe

- **Post-event, aucun rush.** L'event LFD 2026 (4-5 sept) tient **sans GCP** — cf. la décision dédiée.
  La migration n'a **aucune raison d'être faite avant** l'event. On ne se met pas de pression.
- **Ce qui sépare A de B, c'est le CODE (stateless), pas l'infra.** L'infra est commune à ~90 %
  (VPC, Cloud SQL, Nginx, pgBouncer, secrets, CI/CD). Le delta B = **Memorystore + LB + MIG**,
  cheap à poser mais **inutiles tant que le code n'est pas stateless**.
- **Le seul vrai verrou = le chantier clustering / statelessness.** C'est lui qui a fait capoter le
  kickoff Entiovi du 13 avril (WebSocket/sessions en stateless, pooling Prisma). Il faut le faire
  **de toute façon** → autant **viser B directement** et **migrer une seule fois**.

## 1. Décision de cadrage (2026-07-08)

> **🎯 Archi cible = Stratégie B, visée directement. Une seule VM au début (`min=1`), un seul cutover.**
> On **abandonne** l'idée d'une « Stratégie A comme étape livrée séparément » : post-event, elle ne
> fait **gagner aucun temps** (son seul avantage était d'être atteignable sans clustering) et elle
> obligerait à **migrer deux fois**. Comme le clustering doit être fait **de toute façon**, on pose
> l'**infra B**, on fait le clustering tranquillement, et on bascule **une fois**.

**Séquence cible :**

1. **Entiovi pose la fondation infra B** : VPC, **Cloud SQL** (HA/PITR natifs), Nginx/pgBouncer par VM,
   Secret Manager, CI/CD, + **Memorystore** + **template MIG** + **LB régional**.
2. **Nous faisons le chantier clustering** (app stateless) : présence + registry imprimantes en Redis,
   `socket.io-redis-adapter`, zéro état en process, pooling Prisma validé. Sans pression de deadline.
3. **Cutover unique** vers GCP. On tourne en **mono-instance** (`min=max=1`) jusqu'à validation du
   clustering, **puis on allume l'autoscaling** (`min=2`+).

> **🔀 Option de dé-risquage (choix, pas obligation) :** mettre en prod sur la **VM unique d'abord**
> (infra B posée, `min=max=1`, code encore stateful), valider la migration **infra/DB seule**, **puis**
> activer stateless + autoscaling. Ce n'est **pas** « la Stratégie A » — c'est **B en mono-instance**,
> l'autoscaling étant un **interrupteur** qu'on allume quand le clustering est validé. Évite de mélanger
> migration cloud **et** gros refactor dans le même big-bang.

## 2. Rôle d'OVH après la migration complète

Une fois la prod 100 % sur GCP, OVH n'est **jamais** un nœud live (cross-cloud actif-actif = over-engineering, écarté).

| Rôle | Intérêt | Verdict |
|---|---|---|
| **Staging / pré-prod** | env de test isolé, **déjà en place**, pas cher | ✅ **le plus utile** |
| **Backup offsite / cold DR** | dumps Cloud SQL **hors GCP** → protège d'une catastrophe compte/région/facturation | ✅ défense en profondeur |
| **Décommission** | Cloud SQL a déjà HA+PITR ; un projet GCP séparé peut servir de staging | 🟡 légitime si on veut tout centraliser |

> **Défaut recommandé : OVH = staging + cible de backup offsite.** On réutilise l'existant, on gagne un
> filet **cross-cloud**, sans payer un staging GCP en plus. Décommissionner reste valable si on préfère
> centraliser sur GCP.

## 3. Objectifs du call (demain)

- **Court terme (event) :** review du **Gotenberg / Cloud Run déjà déployé** (ingress privé, auth IAM
  service-to-service, egress VPS→GCP, région, cold-start jour-J, coût).
- **Long terme (post-event) :** acter **l'archi cible B visée directement** + le **rôle d'OVH**, et
  surtout obtenir d'Entiovi la **spec « MIG-ready »** (checklist exacte de statelessness) pour **cadrer
  notre chantier clustering**.

Détail des demandes → [02 — questions Entiovi](./02-questions-entiovi.md).

## 4. Dépendances & garde-fous

- ⛔ **Bloquant :** le [chantier clustering](../../en-cours/api-scaling-clustering/README.md) doit être
  livré avant d'activer l'autoscaling (`min≥2`).
- 🔐 **Secrets :** les credentials partagés à Entiovi le 9 avril doivent être **rotés** + passés en
  **Secret Manager** (cf. journal privé).
- 🗓️ **Timing :** migration **post-event**. Ne rien déstabiliser avant le 4-5 sept.
