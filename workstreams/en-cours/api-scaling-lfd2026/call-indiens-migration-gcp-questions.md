# Call équipe Inde — Migration GCP : questions & décision

> **Objet :** préparer le call avec l'équipe (Inde) pour trancher : **migration GCP complète
> faisable d'ici 1 mois ?** Sinon → basculer sur le scénario **« GCP = serveur de secours (DR),
> VPS OVH = primaire »** (inverse de ce qui est décrit dans le PCA aujourd'hui).
> **Date :** 1er juillet 2026 · **Rattachement :** workstream `api-scaling-lfd2026`
> **Réfs :** [plan-action-lfd2026.md](./plan-action-lfd2026.md) (chantier F) ·
> [plan-continuite-activite.md](./plan-continuite-activite.md) (§3 archi, §10 sauvegardes)

---

## Contexte & enjeu (à garder en tête pendant le call)

- **Prod actuelle :** mono-VPS OVH (`51.75.252.74`, Docker Compose : `ems-postgres`, `ems-api`,
  `ems-nginx`, `ems-redis`). **1 seul process Node, Postgres non tuné, pas de réplica, pas de
  CI/CD, pas de monitoring.**
- **Événement :** LFD 2026 / MEAE, ~12 400 inscriptions, **4-5 sept 2026**.
- **Timing :** on est le **1er juillet** → « migration d'ici 1 mois » = **fin juillet**, soit
  **~5 semaines de marge** avant l'event.
- **Plan actuel** (chantier F) : la HA/PITR est **reportée à GCP** en supposant que **GCP devient le
  primaire** (Cloud SQL = HA + PITR natifs).
- **Intuition retenue :** faire un *big-bang* **GCP-primaire ~5 semaines avant un event
  institutionnel MEAE**, sur un environnement jamais testé sous charge = **risque n°1**. Inverser
  (GCP = secours, VPS = primaire) est **plus prudent** et cohérent avec la logique de descope du plan.

---

## Partie 1 — Questions : « migration GCP complète en 1 mois, faisable ? »

### Périmètre & architecture cible
1. Pour **chaque brique** (API NestJS, Postgres, Redis, stockage billets/exports, LB, WAF/anti-bot,
   DNS), vous proposez quoi côté GCP : **managé** (Cloud SQL / Cloud Run / Memorystore / GCS /
   Cloud Armor) ou **lift-and-shift** du Docker Compose sur une VM GCE ?
2. Migration **iso-fonctionnelle** (rejouer le compose tel quel) ou **ré-architecture** vers du
   managé ? (rapide mais peu de bénéfices vs. HA réelle mais plus long)
3. Qui possède le **projet GCP, la facturation, l'org, l'IAM** ? Déjà provisionné ou tout à créer ?

### Migration des données
4. Comment migre-t-on Postgres : **DMS / réplication logique continue** ou **dump/restore** ?
   Quelle **fenêtre de coupure** (downtime) au cutover ?
5. Comment on **valide l'intégrité** post-migration (counts, checksums, cohérence jauges/inscriptions) ?
6. Quel **plan de rollback** si le cutover échoue le jour J ?

### Calendrier & validation
7. Estimation réaliste pour un environnement **testé et validé SOUS CHARGE** (spike 3000 VUs) —
   pas juste « ça tourne » ?
8. Pouvez-vous livrer un **staging GCP d'abord**, qu'on charge (k6) **avant** tout cutover ?
9. Qu'attendez-vous **de nous** (accès repo, env vars, contrôle DNS, secrets) et sous quel délai ?

### Exploitation & astreinte
10. **Monitoring/alerting** GCP (Cloud Monitoring) + **CI/CD** de déploiement : inclus dans le mois
    ou après ?
11. **Qui est d'astreinte pendant l'event 4-5 sept** et **maîtrise l'ops GCP** ? (aujourd'hui
    personne côté équipe ne l'exploite)
12. **Coût mensuel estimé** de la cible (Cloud SQL HA + compute + LB + egress) ?

### Question qui tranche (à poser franchement)
13. « Êtes-vous à l'aise pour un **cutover primaire ~5 semaines avant un event institutionnel
    critique**, ou recommandez-vous de **geler la prod (VPS) pour l'event et migrer après** ? »

> 🔑 **Critère de GO :** GO uniquement si (a) staging GCP + **load test 3000 VUs passé**,
> (b) **rollback prouvé**, (c) **astreinte GCP identifiée**, (d) **freeze ≥ 1 semaine avant l'event**.
> Sinon → **Partie 2**.

---

## Partie 2 — Plan B : GCP = secours (DR), pas primaire

**Pitch :** on garde le **VPS OVH (éprouvé) comme primaire pour l'event**, **GCP devient le plan de
reprise (DR)**. On **inverse le PCA**. Filet de sécurité **sans** risquer le cœur de prod à 5
semaines de l'event, et **GCP devient le tremplin** pour une vraie migration primaire *après*
l'event, faite au calme.

### Trois patterns (du + léger au + costaud — à faire arbitrer par eux)
- **Pilot light :** archivage **WAL + backups en continu vers GCS**, env GCP prêt mais éteint.
  Reprise = promotion à la demande. RTO plus long, coût quasi nul.
- **Warm standby (recommandé) :** **Cloud SQL en réplica externe** du Postgres VPS (réplication
  logique / DMS continu) → GCP a une copie quasi temps réel. API en veille sur Cloud Run/GCE.
  Bascule = promote replica + DNS.
- **Actif/passif complet :** stack GCP tournante en permanence, bascule DNS quasi immédiate.
  Coût le plus élevé.

### Questions pour ce scénario
1. **Cloud SQL peut-il être réplica externe** d'un Postgres auto-hébergé (réplication logique /
   DMS continu) ? Quel **RPO** réel (secondes ? minutes ?) ?
2. Sinon, **WAL archiving continu vers GCS** + capacité à **promouvoir** un env « pilot light » :
   quel **RTO** pour promouvoir (DNS TTL + promote + déploiement API) ?
3. On fait tourner l'API GCP en **warm standby** ou **on la lève à la demande** lors d'un incident ?
4. Comment on **teste la bascule** (drill DR) **sans toucher la prod** ? (indispensable avant l'event)
5. **Coût** d'un warm standby quasi-idle (Cloud SQL replica + compute mini) par mois ?
6. Ce montage DR est-il **le socle d'une future migration primaire** : après l'event, on **promeut
   GCP en primaire** à froid, proprement ?
7. **RTO/RPO garantis** cohérents avec le PCA (RPO ≈ 0, RTO ~5 min) ? (⚠️ réplication logique
   externe ≠ RPO 0 strict)

---

## Impact sur la doc PCA (si plan B retenu)

Inverser **§3 (architecture)** et **§10 (sauvegardes)** du
[plan-continuite-activite.md](./plan-continuite-activite.md) :
aujourd'hui l'archi décrite est *PostgreSQL primaire → réplica* sur une même infra. À reformuler en
**« VPS OVH = primaire de production, GCP/Cloud SQL = site de reprise (DR) »**, avec la procédure de
bascule et le **RPO/RTO réels** obtenus par réplication externe.

---

## Décision (à remplir après le call)

- [ ] Migration GCP-primaire d'ici fin juillet : **GO / NO-GO** → _____
- [ ] Si NO-GO : pattern DR retenu (pilot light / warm standby / actif-passif) → _____
- [ ] RPO / RTO annoncés par l'équipe → _____
- [ ] Coût mensuel → _____
- [ ] Qui d'astreinte event + ops GCP → _____
- [ ] Prochaine action / date drill DR → _____
