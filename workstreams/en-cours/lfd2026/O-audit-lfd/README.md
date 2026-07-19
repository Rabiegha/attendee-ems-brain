# Chantier O — Audit métier et sécurité LFD

> **Périmètre strict :** fournir pour LFD 2026 la journalisation des accès et des actions
> back-office demandée par le CDC. Ce chantier ne construit pas une plateforme générale de logs
> techniques : Sentry, Nginx, PHP, métriques et diagnostic restent dans 0-MON/K.

- **Horizon :** avant LFD, à terminer avant le k6 final bout en bout.
- **Owner recommandé :** Rabie pour le socle ; Corentin contribue au lot `O-BIL`.
- **Estimation brute :** **5–8,5 j-dev**, dont une partie des hooks et tests est réalisée dans BIL/T
  et ne doit pas être comptée deux fois.
- **Source de vérité event :** PostgreSQL `audit_logs`.
- **Statut initial :** audit statique effectué le 19/07 ; implémentation et preuve E2E à faire.

## Pourquoi un chantier autonome

Le socle Attendee existe déjà : table `audit_logs`, intercepteur global, métadonnées JSON et viewer
super-admin. Il ne faut pas le jeter. En revanche, le comportement actuel ne suffit pas comme preuve
CDC : plusieurs actions sont seulement déduites du nom d'une route, les différences avant/après sont
absentes, les échecs de connexion BIL perdent le contexte du navigateur, et le bouton de fermeture
BIL ne correspond pas fidèlement à l'événement enregistré côté Attendee.

O devient propriétaire du **contrat, de la fiabilité, du stockage et de la preuve**. Les chantiers
fonctionnels restent propriétaires du moment où leurs actions sont émises : BIL émet les actions du
back-office, mais n'invente ni format ni stockage d'audit.

## Fondation réutilisable après LFD

Les producteurs utilisent un port `AuditPublisher` et une enveloppe versionnée `AuditEventV1`, sans
dépendre directement de Prisma, MongoDB ou BullMQ. Pour LFD, l'adaptateur écrit dans `audit_logs`.
Après l'événement, P pourra brancher ce même port sur l'outbox de L sans réécrire les producteurs.

Le port doit accepter un contexte transactionnel optionnel, par exemple
`publish(event, { tx })` :

- pour LFD, l'adaptateur PostgreSQL écrit une ligne idempotente, avec index adaptés à
  `lfd_event_id`, `actor_user_id`, `event_name` et `occurred_at` ;
- lorsqu'une transaction métier existe, l'adaptateur utilise cette transaction afin que l'état et
  sa preuve restent cohérents ;
- après LFD, l'adaptateur outbox de L conserve exactement le même appel et écrit l'enveloppe dans
  l'outbox transactionnelle ; les consommateurs de P construisent ensuite les projections et archives.

La scalabilité vient donc de la stabilité du contrat et du remplacement de l'adaptateur, pas du choix
prématuré d'une base. Le stockage PostgreSQL direct est volontairement suffisant pour le trafic
back-office LFD ; tout événement ajouté au chemin public d'inscription doit être mesuré avant d'être
rendu obligatoire.

Champs obligatoires de l'enveloppe :

- `event_id` unique et `schema_version` ;
- `event_name`, `occurred_at` et `result` (`success`/`failure`/`denied`) ;
- snapshot de l'auteur : `actor_user_id`, email, nom et **rôle au moment de l'action** ;
- `org_id` et `lfd_event_id` ;
- `target_type` et `target_id` ;
- `request_id`, IP réelle et user-agent lorsque disponibles ;
- `changes` limité aux champs modifiés avec valeurs avant/après utiles ;
- métadonnées minimisées, sans mot de passe, JWT, token public, pièce jointe ni contenu d'export.

L'intercepteur HTTP générique reste un filet de diagnostic. Il ne remplace jamais un événement
explicite pour une action sensible.

## Événements obligatoires LFD

| Domaine | Événements minimaux | Données spécifiques attendues |
| --- | --- | --- |
| Authentification | `auth.login_succeeded`, `auth.login_failed`, `auth.logout` | email normalisé, raison codifiée, IP réelle, user-agent |
| Autorisation | `authorization.denied` | rôle, permission/action demandée, cible LFD |
| Inscriptions | `registration.updated`, `registration.status_changed`, `registration.deleted` | inscription, champs modifiés, avant/après |
| Rôles | `user.role_changed`, `user.access_revoked` | utilisateur cible, ancien/nouveau rôle, périmètre LFD |
| Sessions | `session.registration_opened`, `session.registration_closed`, `session.capacity_changed`, `session.schedule_changed` | session, état/jauge/horaire avant et après |
| Exports | `registration.export_requested`, `registration.export_completed`, `registration.export_failed` | format, filtres, nombre de lignes, jamais le contenu |

La création publique d'une inscription peut conserver un événement métier explicite, mais ce choix
doit être mesuré sous charge. Il ne faut plus transformer automatiquement tout détail HTTP réussi en
preuve d'audit métier.

## Lots

| Lot | Contenu | Estimation |
| --- | --- | ---: |
| O0 — contrat | `AuditEventV1`, nomenclature, minimisation PII, politique avant/après | 0,5–1 j |
| O1 — socle | port/adaptateur PostgreSQL, idempotence `event_id`, rôle snapshot, alerte d'échec | 1–1,5 j |
| O-AUTH | succès/échec/logout, vraie IP/user-agent BIL, refus d'autorisation | 0,5–1 j |
| O-BIL | inscription, rôle, ouverture/fermeture réelle, jauge/horaire, export | 1,5–2,5 j |
| O-ACCESS | rétention, accès au viewer, procédure de restitution, décision client | 0,5–1 j |
| O-TEST | E2E, absence de secrets, charge audit activé, rapport de preuve CDC | 1–1,5 j |
| **Total brut O** | sans double-compter hooks BIL ni harnais T/K | **5–8,5 j-dev** |

## Fiabilité pragmatique avant LFD

- les événements obligatoires sont explicites et écrits de façon attendue, pas seulement depuis le
  callback `finish` en fire-and-forget ;
- lorsque le service possède déjà une transaction Prisma, écrire l'audit critique dans la même
  transaction si cela reste local et lisible ;
- pour les accès sans transaction métier, attendre l'écriture et alerter sur l'échec ;
- le système d'audit ne doit jamais exposer de secret ni provoquer un contournement RBAC ;
- toute perte ou erreur d'écriture produit une métrique/alerte exploitable dans 0-MON/K ;
- le k6 ciblé L9.b reste exécuté avant O ; le k6 final est rejoué avec la configuration d'audit de
  production afin de mesurer son coût réel et, si applicable, le drainage du backlog.

La transactional outbox **généralisée** reste hors scope avant LFD. Elle appartient à L/P. O prépare
le contrat et les points d'insertion qui permettront cette migration.

## Accès aux journaux

Le CDC exige la journalisation, pas explicitement une page de consultation MEAE. Le viewer actuel
reste réservé aux super-admins tant que le client n'a pas demandé autre chose. Question à trancher :

> Le MEAE souhaite-t-il consulter/exporter lui-même le journal LFD, ou une conservation sécurisée
> restituable sur demande suffit-elle ?

Si un accès client est demandé, ajouter `audit.read:assigned`, limité à l'organisation et à
l'événement LFD, disponible uniquement pour `ADMIN_LFD`. Ne jamais ouvrir le viewer cross-tenant.

## Critères de sortie

- chaque action obligatoire réussie, refusée ou échouée produit exactement l'événement attendu ;
- l'auteur, son rôle au moment de l'action, l'organisation, l'événement, la cible et la date sont
  présents ;
- les changements sensibles ont un avant/après minimisé et lisible ;
- BIL transmet la vraie IP, le user-agent et un `request_id` de confiance ;
- ouvrir/fermer depuis BIL modifie l'état autoritatif Attendee et produit un audit fidèle ;
- aucun mot de passe, JWT, token public ou contenu complet d'export n'est stocké ;
- les permissions du viewer et la durée de conservation sont documentées et testées ;
- les E2E interrogent réellement `audit_logs` et couvrent les trois rôles LFD ;
- le rapport k6 final inclut le coût et la complétude de l'audit activé ;
- une preuve de recette point par point est disponible pour le CDC.

## Note de décision — si k6 se dégrade avec l'audit activé

Un mauvais résultat ne conduit pas à désactiver silencieusement l'audit ni à passer ses écritures en
fire-and-forget. La procédure suivante s'applique.

### 1. Prouver que l'audit cause la régression

Rejouer le même scénario k6 avec et sans audit, sur la même version, les mêmes données et la même
infrastructure. Comparer débit, p95/p99, erreurs HTTP, durée des transactions, verrous PostgreSQL,
connexions DB, CPU/disque et durée des insertions d'audit. Vérifier en parallèle les événements
perdus, dupliqués ou incomplets : une bonne latence avec un audit incorrect reste un échec.

### 2. Optimiser avant de changer l'architecture

Dans cet ordre :

1. supprimer les doublons entre l'intercepteur générique et les événements explicites ;
2. limiter le hot path aux événements réellement obligatoires pour le CDC ;
3. conserver uniquement les champs modifiés dans `changes`, jamais les objets complets ;
4. réduire les métadonnées et vérifier la taille des lignes ;
5. conserver seulement les index nécessaires à la consultation et à la preuve ;
6. garantir exactement un `AuditEventV1` par action métier attendue.

Le trafic back-office doit être audité complètement. La création publique d'une inscription ne doit
pas recevoir un audit lourd par défaut si le CDC ne l'exige pas ; un éventuel événement minimal doit
être décidé et mesuré explicitement.

### 3. Gate de décision recommandé

Les seuils contractuels restent ceux validés pour le test final. Comme budget d'ingénierie provisoire :

- SLO respecté et régression inférieure à environ 5 % : résultat acceptable et documenté ;
- SLO respecté mais régression entre environ 5 et 10 % : optimisation puis nouveau test ;
- SLO dépassé, régression supérieure à environ 10 %, erreurs ou saturation DB : validation bloquée ;
- événement obligatoire perdu, dupliqué ou contenant un secret : validation bloquée quelle que soit
  la performance.

### 4. Option mesurée si PostgreSQL direct reste limitant

Accélérer uniquement une outbox minimale compatible avec L : la transaction métier écrit une petite
enveloppe `AuditEventV1` idempotente dans PostgreSQL, puis un worker construit `audit_logs`. Le port
`AuditPublisher` et les producteurs O-BIL ne changent pas. Cette option anticipe une brique étroite de
L0 ; elle ne déclenche ni la plateforme P complète ni une refonte event-driven générale avant LFD.

À ne pas retenir comme raccourci : publication BullMQ directe après commit sans outbox, erreur
d'écriture ignorée, ajout précipité de MongoDB ou suppression d'un événement CDC obligatoire.
