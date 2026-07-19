# Chantier L — Architecture event-driven Attendee (post-event)

> **Horizon :** post-LFD. La cible est une architecture hybride : commandes métier critiques
> synchrones, événements durables après commit, consommateurs indépendants et idempotents.
> **Impression :** aucun besoin d'impression pour LFD ; le flux `print.*` est exclusivement post-event.

## But final

Découpler les domaines d'Attendee sans transformer chaque appel en événement. La règle est :

- une décision qui doit être connue immédiatement reste synchrone et atomique ;
- un fait métier déjà validé peut produire un événement durable ;
- les effets secondaires lents ou indépendants sont traités par des consommateurs séparés.

## Socle d'architecture cible

1. Transaction métier dans PostgreSQL.
2. Écriture d'un événement dans une **transactional outbox**, dans la même transaction.
3. Relay de l'outbox vers BullMQ ou un bus adapté.
4. Consommateurs avec idempotency key, retry/backoff, dead-letter et observabilité.
5. Inbox/déduplication lorsque le consommateur modifie un état métier.
6. Versionnement des contrats d'événements et politique de rétention/rejeu.

BullMQ reste adapté aux jobs internes et au volume actuel. Le choix d'un broker supplémentaire ne se
fait qu'après mesure d'un besoin de diffusion, de rétention ou de rejeu que BullMQ ne couvre pas.

## Cartographie initiale des flux candidats

| Fait ou commande | Reste synchrone | Candidats asynchrones |
| --- | --- | --- |
| `registration.confirmed` / `waitlisted` / `cancelled` | unicité, capacité, statut DB | billet, email, CRM/n8n, événements d'audit consommés par P, stats, WebSocket |
| `session.checkin.recorded` | validation QR, idempotence, présence DB, réponse agent | stats, présence live, événements d'audit consommés par P, notifications, analytics |
| `event.checkin.recorded` | même garantie de scan | compteurs/analytics et intégrations |
| `ticket.requested` / `ticket.generated` / `ticket.failed` | statut du billet | rendu Gotenberg, stockage R2, email prêt à envoyer |
| `email.requested` / `accepted` / `delivered` / `failed` | choix du template et destinataire | Mailgun, webhooks, métriques, alertes, réconciliation |
| `print.requested` / `completed` / `failed` | autorisation et sélection du badge | rendu, spool, PrintNode, retry contrôlé |
| `attendee.created` / `updated` | validation et persistance | scoring, affinité, tags calculés, CRM, recherche/indexation |
| `invitation.created` / `accepted` / `revoked` | droits et état de l'invitation | email, audit, rappels |
| `event.published` / `closed` / `purged` | transition d'état autorisée | cache, notifications, exports, purge différée |
| `export.requested` / `ready` / `failed` | autorisation et création du statut | génération lourde, stockage, notification |
| `partner.scan.recorded` | validation/idempotence | métriques, intégration partenaire, audit |
| `webhook.received` | signature et persistance minimale | traitement, déduplication, agrégation et projection |

Cette liste est une hypothèse de cadrage issue des modules actuels ; chaque flux devra être validé
avec le métier. Tout n'a pas vocation à devenir asynchrone.

## Découpage recommandé

- **L0 — standards** : nomenclature, enveloppe, correlation/causation IDs, versions, sécurité/PII.
- **L1 — outbox/inbox** : librairie commune, relay, retry, DLQ, dashboard et réconciliation.
- **L2 — inscription/billet/email** : transformer C2.1 en premier flux de référence après LFD.
- **L3 — check-in/live** : projections et effets secondaires, sans différer la validation du scan ;
  le stockage et le viewer d'audit appartiennent à P.
- **L4 — impression** : file durable, connecteur isolé, reprise et déduplication.
- **L5 — intégrations et traitements lourds** : n8n/CRM, exports, scoring, webhooks, purge.

## Estimation initiale

> Estimation en **jours-développeur** pour une migration incrémentale sur BullMQ. Elle n'inclut pas
> l'adoption d'un nouveau broker distribué, qui demanderait un cadrage supplémentaire.

| Lot | Contenu | Estimation |
| --- | --- | ---: |
| L0 | standards, enveloppe, conventions, PII, versions et correlation IDs | 2–3 j-dev |
| L1 | transactional outbox/inbox, relay, déduplication, retry/DLQ et monitoring | 5–8 j-dev |
| L2 | inscription → billet → email comme flux de référence | 3–5 j-dev |
| L3 | check-in et projections live, audit et analytics | 3–5 j-dev |
| L4 | impression durable et connecteur isolé | 3–5 j-dev |
| L5 | invitations, scoring, exports, n8n/CRM, webhooks et purge par incréments | 8–15 j-dev |
| L6 | tests de rejeu, panne, compatibilité de contrats, documentation et runbooks | 3–5 j-dev |
| **Total L** | migration de la cartographie initiale | **27–46 j-dev** |

Prévoir environ **2 à 4 mois calendaires** avec un développeur principal, car chaque lot doit être
mis en production, observé et stabilisé avant le suivant. Il ne faut pas attendre la fin du chantier
pour en tirer de la valeur :

- **socle réutilisable L0+L1 : 7–11 j-dev** ;
- **premier flux de référence L0+L1+L2 : 10–16 j-dev** ;
- les lots L3–L5 peuvent ensuite être priorisés indépendamment selon les incidents et besoins métier.

La fourchette devra être révisée après inventaire précis des producteurs/consommateurs et décision sur
la profondeur de migration de chaque module. Un nouveau broker, du CDC ou une conservation longue des
événements ne sont pas compris dans ces chiffres.

## Frontière avec O et P

- [O — audit LFD](../O-audit-lfd/README.md) livre avant l'événement `AuditEventV1`, les événements
  obligatoires, l'adaptateur PostgreSQL et la preuve CDC, sans outbox généralisée.
- **L0/L1** deviennent après LFD la fondation générique : enveloppe commune, outbox/inbox, relay,
  idempotence, retry et DLQ.
- [P — plateforme de journalisation](../P-plateforme-journalisation/README.md) branche le port O sur
  L0/L1 et possède les projections, sinks, archives, règles de rétention, viewer et logs techniques.

L ne crée donc pas un second format d'audit et P ne crée pas un second mécanisme d'outbox. Les
estimations L0/L1 et P incrémental ne doivent jamais être additionnées deux fois.

## Critères de sortie

- aucun risque « commit DB réussi mais job perdu » sur les flux migrés ;
- événements nommés comme des faits passés et contrats versionnés ;
- consommateurs rejouables et idempotents ;
- traçabilité bout en bout par correlation ID ;
- lag, erreurs, retries et DLQ visibles et alertés ;
- documentation claire des données personnelles et durées de conservation.

## Relation avec F et N

- **N** livre uniquement le découplage réaliste requis pour LFD.
- **F** rend les processus et l'API interchangeables/multi-instance.
- **L** fiabilise et généralise les échanges métier après LFD.

Stateless et event-driven se renforcent, mais restent deux objectifs distincts : F traite l'emplacement
de l'état d'exécution ; L traite la propagation fiable des faits métier.
