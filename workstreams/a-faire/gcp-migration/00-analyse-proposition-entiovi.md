# Analyse — proposition GCP d'Entiovi (« Revised, Nginx-Managed, No GCP LB/CDN »)

> **Réfs :** [README workstream](./README.md) · [archi cible](./01-archi-cible.md) ·
> journal privé `local-files/migration-private/MIGRATION_LOG.md` (chronologie + docs bruts, hors git).

---

## 0. Contexte que le PDF ne dit pas (mais le journal privé si)

Ce PDF **n'arrive pas de nulle part** : c'est la proposition **révisée** d'Entiovi après l'échec du
kickoff.

- **13 avril 2026 — kickoff RATÉ.** Blocages : incompatibilité app actuelle ↔ archi **stateless**
  cible, WebSockets/sessions en stateless, **risque pooling Prisma en serverless**. Kickoff **non validé**.
- **15 avril 2026 — décision d'archi :** **abandon du full serverless** → **archi hybride autoscaling
  (Compute Engine / MIG)**.

➡️ **Ce PDF EST le résultat de cette décision.** Il va dans le bon sens. **Mais le blocage d'avril
(l'app n'est pas stateless) n'a pas disparu** — il est juste **déplacé dans la Stratégie B**. C'est le
fil rouge de toute l'analyse.

## 1. Ce qui est bon pour nous

- ✅ **Cloud SQL managé (HA + PITR natifs)** dans les deux stratégies → c'est **exactement** ce qu'on a
  reporté (chantier F). On l'obtient « gratuitement ». Fin du « travail jeté » de la réplication à la main.
- ✅ **Nginx remplace LB/CDN/Armor** → pragmatique, économe (~47-105 $/mois), garde le contrôle. Raisonnable
  à notre échelle.
- ✅ **Reco Entiovi = commencer petit, monter en charge** → cohérent avec notre lecture (mono-instance
  d'abord, autoscaling ensuite).
- ✅ Leur note « optimistic assumes BullMQ refactoring done » **valide de l'extérieur** notre analyse :
  sans file/queue, un bulk import dégrade les check-ins → c'est notre **L9 + chantier J**.

## 2. Ce qu'il faut challenger au call

- ⚠️ **Capacité annoncée théorique.** Ils parlent de 50-150 check-ins/s sur une VM ; **nous avons mesuré
  ~33/s** bloqué par la **transaction d'inscription**. Le plafond est **algorithmique, pas matériel** :
  changer d'infra **sans L9** ne donnera pas ces chiffres. À poser clairement.
- ⚠️ **Sécurité périmètre.** Pas de WAF/anti-DDoS entreprise (Nginx + fail2ban). **Non bloquant pour
  l'event** (event = OVH, GCP pas utilisé). Reste un détail **post-event / prod GCP** de faible priorité.
- ⚠️ **Dimensionnement Cloud SQL** (vCPU/RAM/stockage) à cadrer vs ~12 400 inscrits + pic 3000 concurrents.

## 3. Le point capital : A vs B, c'est le CODE

| | **Stratégie A** (single VM) | **Stratégie B** (MIG autoscaling) |
|---|---|---|
| Topologie | ≈ notre setup actuel (Docker Compose) **mais DB managée** | N VMs derrière LB régional |
| Besoin **clustering/stateless** | ❌ aucun | ✅ **obligatoire** (Socket.IO Redis adapter, Memorystore, registry imprimantes partagé, zéro état en process) |
| Livre HA/PITR ? | ✅ (Cloud SQL) | ✅ (Cloud SQL) |
| Faisable sans clustering | ✅ | ❌ (= le blocage d'avril) |

**Lecture :** l'infra est commune à ~90 %. Le **seul delta B** = Memorystore + LB + MIG, **inutiles tant
que le code n'est pas stateless**. Donc **le verrou n'est pas l'infra, c'est le [chantier clustering](../../en-cours/api-scaling-clustering/README.md)** — qu'il faut faire **de toute façon**.

## 4. Décision de séquencement

> **On abandonne « A comme étape livrée séparément ».** Post-event, elle ne fait gagner **aucun temps**
> (son seul atout = atteignable sans clustering) et obligerait à **migrer deux fois**. On **vise B
> directement**, mono-instance au début, **un seul cutover**. Détail → [archi cible](./01-archi-cible.md).

Option de dé-risquage possible (mono-instance d'abord, autoscaling ensuite) : cf. README §1.
