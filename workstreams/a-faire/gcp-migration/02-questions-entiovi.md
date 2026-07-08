# Questions & demandes pour Entiovi (call)

> **Réfs :** [README workstream](./README.md) · [analyse Entiovi](./00-analyse-proposition-entiovi.md) ·
> [archi cible](./01-archi-cible.md).
> **Objectif du call :** (court terme) review Gotenberg/Cloud Run · (long terme) acter l'archi cible B
> + obtenir la **spec « MIG-ready »** pour cadrer notre chantier clustering.

---

## A. Court terme — Gotenberg / Cloud Run (déjà déployé)

- [ ] Le service est-il en **ingress privé** (pas exposé publiquement) ?
- [ ] **Auth service-to-service** : IAM / clé ? Comment le VPS OVH s'authentifie ?
- [ ] **Egress VPS→GCP** sécurisé (pas d'exposition Gotenberg au monde) ?
- [ ] **Région** + **cold-start** : comportement jour-J (garder un min d'instances chaudes) ?
- [ ] **Coût** réel pay-per-use estimé sur le volume event ?

## B. Long terme — archi cible & séquencement

- [ ] On **vise la Stratégie B directement** (pas d'étape A séparée), **mono-instance au début**,
      **un seul cutover**. **OK de votre côté ?**
- [ ] Répartition : **vous** = infra GCP (VPC, Cloud SQL, Memorystore, MIG, LB, Secret Manager, CI/CD).
      **Nous** = code stateless (clustering). Confirmé ?
- [ ] ⭐ **Spec « MIG-ready »** : donnez-nous la **checklist exacte** de ce que l'app doit satisfaire pour
      être clusterisable — Socket.IO Redis adapter, registry imprimantes en Redis, zéro état en process,
      **pooling Prisma** (le risque que vous aviez soulevé). → c'est ce qui **cadre notre chantier clustering**.

## C. Migration des données & cutover

- [ ] OVH Postgres → **Cloud SQL** : **dump/restore** ou **DMS** ? Fenêtre de coupure ? Plan de bascule/rollback ?
- [ ] **Dimensionnement Cloud SQL** (vCPU/RAM/stockage) vs ~12 400 inscrits + pic 3000 concurrents ?
- [ ] **HA + PITR** Cloud SQL : configuration cible (rétention backups, fenêtre PITR) ?

## D. Rôle d'OVH & sécurité

- [ ] Acter **OVH = staging + backup offsite** (pas un nœud live). D'accord ?
- [ ] 🔐 **Rotation** des credentials partagés le 9 avril + passage en **Secret Manager**. À faire ensemble.
- [ ] Sécurité périmètre post-event (WAF/DDoS) : nécessaire pour ce client, ou Nginx + fail2ban suffisent ?

## E. Timing

- [ ] Confirmer : migration **post-event** (après 4-5 sept). Rien ne déstabilise l'event.
- [ ] Jalons proposés côté Entiovi (fondation infra → data → cutover) ?
