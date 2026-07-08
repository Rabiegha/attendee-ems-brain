# Workstream — Résilience event LFD 2026 (checklist des protections)

> **Objet :** le **dernier filet avant l'event** — s'assurer que **tout ce qui protège** est réellement
> en place et **testé**, pas juste planifié. Ce workstream ne construit pas de nouvelle feature : il
> **vérifie** que les protections des autres chantiers (débit, monitoring, backup, recovery) sont
> **actives, mesurées et documentées**. À passer en revue **avant J-7**.
> **Date :** 8 juillet 2026 · **Rattachement :** workstream `lfd2026` · **Statut :** 🔴 à vérifier
> **Réfs :** [00-plan-action.md](../../en-cours/lfd2026/00-plan-action.md) (chantier K) ·
> [décision — tenir l'event sans GCP](../../en-cours/lfd2026/decisions/tenir-event-sans-gcp.md) ·
> [02 — capacité live forte charge](./02-capacite-live-forte-charge.md) *(voisin ws sessions)* ·
> [plan de continuité d'activité](../../en-cours/infra-scaling-pca/plan-continuite-activite.md)

---

## 0. Principe

**On ne protège pas contre tout — on protège contre les risques PROBABLES, et on assume le reste.**
La HA (rester debout si la machine meurt) est **reportée à GCP** : c'est le risque le **moins probable**
sur 2 jours, et **assumé**. Les trois vrais dangers — **saturation, disque plein, perte de données** —
se règlent **sur le VPS actuel**. Ce workstream est la **checklist de vérification** de ces protections.

### 🗺️ Carte des risques (rappel)

| Risque | Probabilité | La HA aide ? | Parade (ce workstream vérifie qu'elle est en place) |
|---|---|---|---|
| **Saturation CPU/DB** (3000 concurrents) | élevée | ❌ | L9 + file BullMQ + portier Redis (chantiers I / J) |
| **Disque plein** → API down | moyenne-élevée | ❌ | 0-MON alerte disque + capacité + rotation logs + offload R2 |
| **Perte / corruption données** | moyenne | ❌ | backup offsite + **restore testé** (chantier E) |
| **Machine meurt** (hardware) | faible | ✅ | *(HA → GCP, risque assumé — recovery manuel)* |

---

## 1. Checklist — Saturation (débit)

> Protège du danger n°1 : le pic d'inscriptions qui sature CPU/DB.

- [ ] **L9 codé et mergé** (`feat/register-tx-slim`) — transaction courte + compteur dénormalisé.
- [ ] **Test de non-régression L9** — anti-survente préservé (pas de double-booking sous charge).
- [ ] **File BullMQ** en place pour absorber le burst d'inscriptions (personne rejeté).
- [ ] **Portier Redis atomique** actif sur les sessions à capacité serrée (anti-survente).
- [ ] **Load test combiné** (inscriptions + check-ins) passé sur staging + k6, **au débit cible**.
- [ ] **Cible event chiffrée définie** (ex. « 3000 inscriptions drainées en < X min ») — sinon pas de seuil.

## 2. Checklist — Disque plein (le mode de panne sous-estimé)

> Disque plein → Postgres ne peut plus écrire le WAL → **API down**. La HA n'aide pas.

- [ ] **Alerte disque 70 / 80 / 90 %** active (SMS/Slack) — *la protection n°1*.
- [ ] **Marge de capacité** dimensionnée pour l'event (volume estimé : ~12 400 inscrits + scans + logs + cache PDF), disque provisionné **bien au-dessus**.
- [ ] **Rotation des logs** configurée (`logrotate` + Docker `max-size`/`max-file`) — couper le `console.log` par inscription en prod ou le borner.
- [ ] **Gestion WAL** : `max_wal_size` borné + **surveillance des slots de réplication bloqués** (WAL jamais recyclé = disque qui se remplit seul).
- [ ] **Offload de la donnée lourde hors VPS** : PDF badges → **R2**, backups → **R2 offsite** (pas sur le disque local).
- [ ] **Autovacuum réglé** (limiter le bloat MVCC sous pic d'`UPDATE`).
- [ ] **Redis** : mémoire surveillée + politique si backlog de file gonfle.
- [ ] **Runbook « libérer / redimensionner le volume à chaud »** écrit (OVH permet souvent le resize live).

## 3. Checklist — Perte / corruption de données

> Le vrai risque institutionnel MEAE. Couvert par le backup, **pas** par la HA.

- [ ] **Backup automatique** (cron, fenêtre creuse) — chantier E.
- [ ] **Offsite chiffré** (push de chaque dump vers R2, chiffré au repos).
- [ ] **Restore TESTÉ** : au moins **une restauration prouvée** sur staging + **procédure documentée**.
- [ ] **PITR léger** (archivage WAL sur le VPS) si le temps le permet — « remonter le temps » avant un `DELETE` accidentel.
- [ ] **Alerte si un dump échoue** (un backup silencieusement cassé = faux filet).

## 4. Checklist — Monitoring & visibilité (0-MON)

> Sans ça, on vole à l'aveugle : on découvre les pannes par les plaintes clients.

- [ ] **Uptime externe + alerte** (`/health`) → SMS/Slack si down.
- [ ] **Error-tracking** (Sentry) back + front — exceptions remontées.
- [ ] **Métriques système** (CPU/RAM/**disque**) + dashboard.
- [ ] **Alerting seuils** : CPU élevé, **file BullMQ qui gonfle**, **disque ≥ 70/80/90 %**.
- [ ] **Astreinte / qui est alerté ?** défini pour les 2 jours d'event (numéro joignable jour-J).

## 5. Checklist — Recovery (le plan B assumé)

> Puisqu'il n'y a pas de HA, la **rapidité et la clarté** de la récupération sont la protection.

- [ ] **Procédure de recovery écrite ET testée** : monter un VPS + restaurer backup + re-pointer l'IP/DNS.
- [ ] **RTO estimé** connu (combien de temps pour être de nouveau debout ?) et **acceptable** documenté.
- [ ] **Accès/credentials** de secours accessibles jour-J (pas seulement sur la machine qui peut mourir).
- [ ] **Snapshot / image récente** prête pour reprovisionner vite si besoin.

## 6. Décision de dépendance

- [ ] **SLA MEAE vérifié** : **pas** de SLA d'uptime contractuel strict → décision « pas de HA event » **validée**. Si SLA strict → escalader (GCP accéléré ou failover OVH temporaire pour les 2 jours).

---

## 7. Passage en revue final (avant J-7)

Ce workstream est **VERT** quand :
1. Saturation → **L9 mergé + load test combiné au vert**.
2. Disque → **alerte disque active + logs tournés + offload R2 + capacité dimensionnée**.
3. Perte → **backup offsite + restore prouvé sur staging**.
4. Monitoring → **uptime + Sentry + alerting seuils + astreinte définie**.
5. Recovery → **procédure testée + RTO acceptable**.
6. SLA → **confirmé pas de contrainte contractuelle** (ou escaladé).

> Tant qu'une case critique (1, 2, 3) est rouge, **on n'est pas prêt pour l'event**, quel que soit
> l'avancement des features.
