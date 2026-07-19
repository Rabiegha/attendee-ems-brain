# Chantier P — Plateforme de journalisation Attendee (post-event)

> **Horizon ferme :** entièrement post-LFD. P industrialise l'audit et les logs techniques sans les
> mélanger fonctionnellement. Il réutilise le contrat livré par O et le socle event-driven de L.

- **Estimation incrémentale après L0/L1 :** **15–25 j-dev**.
- **Estimation autonome si L0/L1 n'est pas disponible :** **22–36 j-dev**.
- **Prérequis :** retour d'expérience O, volumétrie réelle, politique RGPD/rétention et L0/L1.
- **Aucun choix MongoDB, OpenSearch ou ClickHouse avant mesure.**

## But final

Construire deux pipelines corrélés mais séparés :

1. **audit métier/sécurité** : complet, non échantillonné, durable, restituable et soumis à RBAC ;
2. **logs techniques/traces** : diagnostic, recherche, alertes, rotation et échantillonnage possible.

Ils partagent `request_id`, `correlation_id` et `causation_id`, mais pas les mêmes règles d'accès, de
rétention, d'immutabilité ni nécessairement le même stockage.

## Architecture cible

```text
transaction métier PostgreSQL
  ├─ état métier
  └─ outbox versionnée (socle L)
          ↓ relay durable
       BullMQ / bus mesuré
          ↓
     consommateurs idempotents
       ├─ projection audit consultable
       ├─ archive audit immuable
       └─ métriques/alertes

API / PHP / workers / Nginx
          ↓ logs et traces structurés
       pipeline technique séparé
```

## Frontières avec les autres chantiers

- **O** définit `AuditEventV1`, couvre le CDC LFD et fournit le retour d'expérience réel.
- **L** possède les standards génériques, l'outbox/inbox, le relay, l'idempotence, le retry et la DLQ.
- **P** possède les projections d'audit, le stockage, la recherche, la rétention, l'archive, le viewer
  et le pipeline de logs techniques.
- **0-MON/K** restent la solution opérationnelle minimale pendant LFD ; P les consolide ensuite.
- **F** traite stateless/multi-instance/HA et ne doit pas être absorbé par P.

Ainsi, P ne crée ni second format d'événement ni second mécanisme d'outbox. Les jours L0/L1 ne sont
pas comptés une seconde fois dans l'estimation incrémentale de P.

## Lots post-event

| Lot | Contenu | Estimation incrémentale |
| --- | --- | ---: |
| P0 — retour O | volumétrie, incidents, qualité et coûts observés pendant LFD | 1–2 j |
| P1 — audit durable | branchement `AuditPublisher` sur outbox L, consumer idempotent, replay/DLQ | 3–5 j |
| P2 — stockage | benchmark et choix du read model, index/partitions, archive immuable | 2–4 j |
| P3 — accès | viewer org/event-scoped, RBAC, export signé, preuve de consultation | 2–4 j |
| P4 — logs techniques | collecte structurée API/PHP/workers/Nginx, corrélation et recherche | 3–5 j |
| P5 — gouvernance | rétention, purge, legal hold, pseudonymisation, sauvegarde/restauration | 2–3 j |
| P6 — validation | panne/rejeu, charge, sécurité cross-tenant, runbooks et SLO | 2–3 j |
| **Total P après L0/L1** | livraison incrémentale | **15–25 j-dev** |

## Décision de stockage

MongoDB est une option à évaluer, pas une cible décidée. Le choix se fera selon :

- volume d'écriture et durée de conservation ;
- types de recherches et agrégations ;
- exigences d'immutabilité et d'archive ;
- coût opérationnel, sauvegarde/restauration et compétences équipe ;
- cloisonnement multi-tenant et suppression RGPD ;
- besoin réel de replay ou d'analytics.

À comparer au minimum : PostgreSQL/JSONB partitionné, ClickHouse pour l'analytique, OpenSearch pour la
recherche et stockage objet immuable pour l'archive. Le pipeline reste indépendant du sink retenu.

## Critères de sortie

- aucun événement audité perdu entre commit métier et projection ;
- rejeu sans doublon grâce à `event_id`/inbox ;
- contrats versionnés et compatibles ;
- séparation effective des permissions audit et logs techniques ;
- recherche cloisonnée par organisation/événement ;
- rétention, purge, archive et restauration prouvées ;
- panne d'un consumer, backlog et reprise testés sous charge ;
- coût et SLO du pipeline documentés ;
- aucun producteur O réécrit lors du passage PostgreSQL direct → outbox L.
