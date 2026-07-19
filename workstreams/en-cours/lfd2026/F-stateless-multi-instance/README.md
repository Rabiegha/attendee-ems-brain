# Chantier F — API stateless, multi-instance et haute disponibilité (post-event)

> **Horizon :** après LFD 2026. Ce chantier ne doit pas détourner l'équipe des garanties nécessaires
> à l'événement. Avant LFD, on cartographie seulement les dépendances et on évite d'ajouter de l'état local.
> **Impression :** hors scope LFD ; l'état PrintNode/local est traité ici uniquement post-event.

## But final

Rendre Attendee déployable sur plusieurs instances interchangeables : une requête peut arriver sur
n'importe quelle instance sans perdre une session, un socket, un fichier, un job ou une impression.
Ce chantier est le prérequis applicatif du scaling horizontal et complète la HA/PITR fournie par GCP.

## État connu

- l'API n'est pas stateless aujourd'hui ;
- le processor email BullMQ tourne encore dans le processus API ;
- les WebSockets ne disposent pas encore d'une stratégie multi-instance finalisée ;
- l'impression et certains fichiers temporaires/artefacts doivent être inventoriés ;
- la HA applicative est reportée, tandis que backup offsite et restauration sont déjà couverts par E/K.

## Inventaire à mener

| Zone | Question à trancher | Cible |
| --- | --- | --- |
| Auth/session | état en mémoire, cookies, refresh tokens, affinité éventuelle | session partagée ou tokens vérifiables par toute instance |
| WebSockets | rooms, présence, émissions locales | adapter Redis, messages partageables, reconnexion testée |
| BullMQ | processors chargés avec l'API | rôles `api` et `worker` séparés, déployables indépendamment |
| Impression | files, imprimantes, état PrintNode/local | queue partagée, jobs idempotents, connecteur d'impression isolé |
| PDF/fichiers | temporaires, cache, stockage local | stockage objet/R2, aucun artefact requis sur disque API |
| Cache | caches mémoire et invalidation | Redis ou cache reconstructible sans affinité |
| Cron/schedulers | risque d'exécution N fois | scheduler unique ou verrou distribué |
| Upload/export | traitements longs et fichiers locaux | stockage partagé + workers dédiés |
| DB | nombre de connexions multiplié par instance | budget global, pgBouncer, limites par rôle |
| Config/secrets | divergence entre instances | configuration centralisée et versionnée |

## Phases

1. **Cartographie** : recenser tout état local et chaque dépendance à une instance précise.
2. **Séparation des rôles** : API HTTP, workers, schedulers et connecteurs d'impression.
3. **Externalisation de l'état** : Redis, PostgreSQL et stockage objet selon la nature des données.
4. **Compatibilité multi-instance** : WebSocket Redis adapter, idempotence, pool DB et arrêt gracieux.
5. **Validation** : deux instances sans sticky session, perte/recréation d'une instance, jobs en cours,
   reconnexion socket, impression, uploads et test de charge.
6. **HA/GCP** : load balancer, health checks, Cloud SQL HA/PITR et procédures de bascule.

## Estimation initiale

> Estimation en **jours-développeur**, à recalibrer après la cartographie. Elle suppose la réutilisation
> de Redis, BullMQ, pgBouncer, R2 et des fondations GCP déjà prévues, sans réécriture fonctionnelle.

| Phase | Contenu | Estimation |
| --- | --- | ---: |
| F0 | audit de l'état local, dépendances, diagramme et plan de migration | 2–3 j-dev |
| F1 | séparation API / workers / schedulers / impression | 2–4 j-dev |
| F2 | externalisation sessions, caches, fichiers et artefacts | 3–6 j-dev |
| F3 | WebSocket Redis adapter, arrêt gracieux, pools et idempotence multi-instance | 3–5 j-dev |
| F4 | tests à deux instances, perte d'un nœud, charge et runbooks | 2–4 j-dev |
| F5 | load balancer, Cloud SQL HA/PITR, déploiement et exercice de bascule | 3–5 j-dev |
| **Total F** | application stateless jusqu'à la HA validée | **15–27 j-dev** |

Avec un développeur principal et les validations nécessaires, prévoir environ **4 à 7 semaines
calendaires**. Le premier jalon utile, « API et workers séparés + état local inventorié », représente
**4 à 7 j-dev** ; il apporte déjà de la valeur sans attendre toute la migration GCP.

La fourchette haute s'applique si l'authentification, l'impression ou les fichiers reposent fortement
sur l'instance locale. La cartographie F0 doit produire une estimation révisée avant engagement.

## Critères de sortie

- aucune donnée métier durable uniquement en mémoire ou sur le disque d'une instance API ;
- API et workers démarrent avec des rôles séparés ;
- deux instances servent le même trafic sans affinité obligatoire ;
- disparition d'une instance sans perte ni double traitement ;
- connexions DB/Redis bornées et observées ;
- runbooks de déploiement, rollback et bascule testés.

## Hors périmètre avant LFD

Pas de clustering « au cas où ». Le déclencheur reste une mesure insuffisante après L9 et les tests
combinés. Les améliorations event-critiques sont suivies dans le chantier N.
