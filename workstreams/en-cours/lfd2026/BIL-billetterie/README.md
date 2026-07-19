# Chantier BIL — Plateforme billetterie

> Gestion billetterie LFD + création de landing pages + **backoffice client**.
> ⚠️ Le **MVP LFD** est avancé ; la plateforme billetterie générique reste un sujet post-event.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-BIL)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **BIL**)
- **Owner :** Corentin — **Statut :** 🟡 chantier global ~35 % (réévalué le 19/07 après ajout du dashboard CDC)

## Lecture du 35 %

Le socle utile LFD 2026 est déjà matérialisé dans `back-office-event` :

- billetterie publique one-page ;
- agenda 2 jours ;
- back-office client ;
- synchronisation des sessions vers Attendee ;
- inscription publique via Attendee ;
- jauges live / fallback local.

Mais le chantier global est réévalué à **~35 %** : le socle existe, mais la partie livrable
client n'est pas encore fermée tant que l'environnement cible, la finition UX/admin, le
durcissement du parcours, le RBAC LFD et le raccord billet PDF + email ne sont pas faits.

Ce n'est donc **pas** 70 % d'une plateforme billetterie générique complète avec offres, paiement,
builder multi-event avancé et self-service post-event. Pour ce périmètre long terme, il faudra
re-cadrer après l'événement.

## Reste à faire MVP LFD

- environnement cible / secrets propres ;
- finition admin/client ;
- RBAC LFD administrateur/opérateur/lecteur, contrôlé côté serveur et reflété dans l'UI ;
- émission des actions back-office vers le contrat [O — audit LFD](../O-audit-lfd/README.md) ;
- raccord billet PDF + email via C2.1/B ;
- durcissement du parcours d'inscription ;
- dashboard des entrées par salle/créneau + export présent/absent via `J-ENTREES` ;
- vérification cahier des charges complet avec le client.

## BIL-ENTREES — vue opérationnelle demandée par le CDC

> **Owner : Corentin**, en intégration avec
> [J-ENTREES](../J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd). Ce n'est pas
> un chantier autonome : H/L9.1 fournit les compteurs, J le snapshot/live et BIL l'expérience client.

### État vérifié le 19/07

- `registered_count` et les jauges de la landing suivent les **réservations**, pas les entrées ;
- le front Attendee sait afficher `present` quand on ouvre une session, mais ne se rafraîchit pas
  après les scans d'un autre appareil ;
- `back-office-event` ne possède pas de dashboard global d'occupation ;
- l'export Attendee Sessions contient email et heures de scan, mais n'inclut pas les confirmés sans
  scan dans la feuille de leur session et ne produit pas de statut explicite `Présent`/`Absent`.

### UI LFD minimale

- une ligne/carte par couple salle + créneau : état, présents maintenant, entrés au moins une fois,
  capacité physique et taux de remplissage ;
- tri par heure et salle, avec seuils visuels sobres (normal, vigilance, plein) ;
- horodatage « mis à jour à » et état de connexion ; si le live est perdu, afficher le dernier
  snapshot sans prétendre qu'il est actuel puis retenter ;
- action d'export post-event avec email, statut d'inscription et statut de participation explicite ;
- aucun détail nominatif envoyé par WebSocket : la liste détaillée et l'export restent des appels
  authentifiés et autorisés côté serveur.

### Droits

- `ADMIN_LFD` et `OPERATOR_LFD` voient le dashboard ;
- `READER_LFD` voit les agrégats en lecture seule ;
- le droit d'export du lecteur reste à faire confirmer au MEAE, car le fichier contient des emails ;
- masquer le bouton selon le rôle ne suffit pas : le proxy PHP et l'API Attendee doivent refuser
  l'export non autorisé et O doit auditer l'action.

### Définition de fini

- les scans réalisés sur plusieurs téléphones convergent vers les mêmes chiffres sur le dashboard ;
- le taux utilise `checkin_capacity ?? capacity` et non la capacité d'inscription seule ;
- un reconnect/rechargement reprend un snapshot serveur exact ;
- l'export contient les confirmés scannés `Présent`, les confirmés non scannés `Absent` et distingue
  les walk-ins, waitlisted, cancelled et blocked ;
- le scénario est inclus dans la répétition K et la matrice de droits BIL-RBAC.

Effort BIL incrémental : **~0,5–1 j-dev** après livraison du contrat J-ENTREES. L'estimation globale
BIL passe de **~6,5–10 j à ~7–11 j-dev** ; la partie backend J/H n'est pas recomptée ici.

## BIL-RBAC — acces multi-utilisateurs demande par le CDC

> **Owner : Corentin.** Le CDC demande un accès simultané avec gestion des droits pour trois profils :
> administrateur, opérateur et lecteur. Il ne demande pas explicitement un éditeur de permissions
> arbitraires utilisable par le client.

### Décision de périmètre

Créer dans Attendee trois rôles dédiés, limités à l'événement LFD via l'organisation et/ou
`EventAccess` :

- `ADMIN_LFD` — Administrateur LFD ;
- `OPERATOR_LFD` — Opérateur LFD ;
- `READER_LFD` — Lecteur LFD.

Le MEAE doit pouvoir inviter un utilisateur et lui attribuer l'un de ces trois rôles. Pour le MVP,
il ne peut pas modifier la matrice interne des permissions : celle-ci est préparée et testée par
Attendee. Cela répond à la « gestion des droits » tout en évitant une configuration dangereuse.

Ne pas transformer tout utilisateur authentifié en administrateur. Aujourd'hui,
`back-office-event/admin/login.php` pose `ADMIN_SESSION_KEY=true` dès qu'Attendee renvoie un token :
c'est une authentification binaire, pas du RBAC.

### Matrice fonctionnelle recommandée

| Capacité BIL LFD | Administrateur LFD | Opérateur LFD | Lecteur LFD |
| --- | :---: | :---: | :---: |
| Voir agenda, espaces, créneaux et jauges | ✅ | ✅ | ✅ |
| Voir inscrits et statistiques | ✅ | ✅ | ✅ |
| Exporter la liste des inscrits | ✅ | ✅ | ✅ si validé MEAE |
| Ouvrir/fermer manuellement un créneau | ✅ | ✅ | ❌ |
| Modifier horaires, capacité et contenu d'un créneau | ✅ | ❌ par défaut | ❌ |
| Créer/supprimer espaces, jours et créneaux | ✅ | ❌ | ❌ |
| Publier/re-publier la configuration | ✅ | ❌ | ❌ |
| Corriger une inscription | ✅ | ✅ si besoin opérationnel | ❌ |
| Supprimer une inscription | ✅ | ❌ par défaut | ❌ |
| Inviter un utilisateur LFD | ✅ | ❌ | ❌ |
| Attribuer `ADMIN_LFD` / `OPERATOR_LFD` / `READER_LFD` | ✅, sans s'auto-élever | ❌ | ❌ |
| Modifier la matrice de permissions | ❌ MVP | ❌ | ❌ |

Les deux arbitrages à faire valider par le MEAE sont l'export pour le lecteur et la correction
d'inscription pour l'opérateur. Appliquer le moindre privilège tant qu'ils ne sont pas tranchés.

### Traduction dans les permissions Attendee

Réutiliser le moteur RBAC Attendee, ses scopes et les accès assignés à l'événement. Base indicative :

- commun : `events.read:assigned`, `registrations.read:assigned` ;
- administrateur : `events.update`, `events.publish`, gestion des inscriptions et attribution des
  trois rôles LFD dans le périmètre autorisé ;
- opérateur : lecture + permissions opérationnelles ciblées d'ouverture/fermeture et, si validé,
  `registrations.update`/export ;
- lecteur : lecture seule, sans aucune permission `create`, `update`, `delete`, `publish` ou `manage`.

⚠️ Avant implémentation, vérifier les clés exactes dans le registry/seeder et les guards des endpoints.
Les sessions ne semblent pas disposer aujourd'hui d'une famille de permissions aussi explicite que
`events.*`/`registrations.*`. Si ouvrir/fermer, modifier et supprimer une session utilisent tous une
permission trop large comme `events.update`, ajouter des permissions ciblées (`sessions.read`,
`sessions.operate`, `sessions.manage`) plutôt que de sur-donner des droits à l'opérateur.

### Contrôles serveur obligatoires dans `back-office-event`

1. Après login, récupérer les grants effectifs et l'accès à l'événement LFD depuis Attendee ; ne pas
   faire confiance à un rôle envoyé par le navigateur.
2. Refuser le login BIL si l'utilisateur ne possède aucun des trois rôles/access LFD.
3. Conserver en session serveur l'identité et les grants nécessaires, avec expiration alignée au JWT.
4. Protéger chaque action serveur : lecture, sauvegarde, ouverture/fermeture, publication, export et
   administration des utilisateurs.
5. Dans `save.php`, autoriser par action et pas seulement avec `ADMIN_SESSION_KEY`. Un refus API
   Attendee ne suffit pas, car le fichier local ne doit pas être modifié par un lecteur.
6. Émettre succès et refus via le lot
   [O-BIL](../O-audit-lfd/README.md#lots), avec utilisateur, rôle, événement, cible, résultat,
   horodatage et avant/après minimisé. O reste propriétaire du contrat et du stockage.
7. Tester qu'un rôle ne peut pas contourner l'UI en appelant directement l'endpoint PHP.

### Comportement de l'interface selon le rôle

- **Administrateur** : interface complète, y compris paramétrage et gestion des accès LFD.
- **Opérateur** : tableau de bord opérationnel ; boutons ouvrir/fermer visibles ; éditeur structurel,
  suppression et gestion des utilisateurs absents ou désactivés.
- **Lecteur** : interface réellement en lecture seule ; champs non éditables et aucun bouton de
  sauvegarde/publication/suppression. Ne pas seulement cacher les boutons : le serveur refuse aussi.
- afficher le rôle actif et le périmètre « LFD 2026 » dans l'en-tête ;
- sur `403`, expliquer « droit insuffisant » sans révéler de données sensibles ;
- si les permissions changent pendant une session, elles doivent être reprises au plus tard au
  renouvellement/rafraîchissement de session, idéalement par revalidation serveur sur action sensible.

### Tests de recette RBAC

- trois comptes de recette, un par rôle ;
- matrice allow/deny testée côté PHP et côté API Attendee ;
- tentative directe sur `save.php` avec lecteur → `403` et aucun fichier modifié ;
- opérateur capable d'ouvrir/fermer sans pouvoir supprimer/reconfigurer ;
- administrateur capable d'attribuer les trois rôles sans élévation hors LFD ;
- travail simultané des trois comptes et journalisation des actions, validée dans O-TEST ;
- expiration/révocation du token et retrait d'un accès testés.

### Estimation ajoutée

| Lot | Estimation |
| --- | ---: |
| Créer/configurer les trois rôles Attendee et scopes LFD | 0,5–1 j |
| Endpoint/résolution des grants pour `back-office-event` | 0,5–1 j |
| Guards serveur PHP par action | 0,5–1 j |
| Adaptation UI par rôle | 0,5–1 j |
| Tests E2E allow/deny des trois rôles | 0,5–1 j |
| **Sous-total BIL-RBAC** | **2,5–5 j-dev** |

La fourchette haute s'applique si des permissions `sessions.*` ciblées doivent être ajoutées au backend
Attendee. Avec BIL-ENTREES, l'estimation globale BIL passe d'environ **4–5 j à 7–11 j-dev**, dont ~3 j
sont déjà consommés ; reste global indicatif **~4–8 j-dev** hors dépendances C2.1/B.

## Frontière BIL / O — pas de sous-chantier dupliqué

`BIL-AUDIT` n'est pas créé comme troisième chantier. Il est remplacé par le lot `O-BIL` :

- **BIL/Corentin** possède les parcours, les guards et les points d'émission au bon moment ;
- **O/Rabie** possède `AuditEventV1`, l'adaptateur PostgreSQL, la fiabilité, la politique de données,
  les E2E d'audit et la preuve CDC ;
- **T/K** réutilisent les scénarios et le rapport, sans recompter l'implémentation ;
- le temps des hooks BIL inclus dans O n'est pas ajouté une seconde fois à l'estimation BIL-RBAC.

Le changement futur vers l'outbox L/P ne doit nécessiter aucune réécriture de l'UI ou des règles RBAC
BIL : seuls le port et son adaptateur évoluent.
