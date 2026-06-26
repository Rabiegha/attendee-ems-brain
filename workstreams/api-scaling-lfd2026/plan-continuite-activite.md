# Plan de Continuité d'Activité — Solution de billetterie Attendee

## 1. Objectif du PCA

Le présent Plan de Continuité d'Activité a pour objectif d'assurer la continuité des services critiques de la solution Attendee pendant la période de préparation, d'ouverture des inscriptions et de déroulement de l'événement.

Les services couverts sont :

- la billetterie numérique en ligne ;
- l'ouverture et la fermeture automatique des créneaux d'inscription ;
- la gestion des jauges par espace, activité et créneau ;
- l'émission des billets numériques avec QR code ;
- le contrôle d'accès par application de scan ;
- le back-office de pilotage ;
- l'export des données d'inscription et de présence.

Le PCA vise à garantir la disponibilité du service, à limiter les interruptions, à préserver l'intégrité des données et à permettre une reprise rapide en cas d'incident.

---

## 2. Services critiques et niveau de priorité

| Service | Criticité | Objectif |
|---|---|---|
| Page publique d'inscription | Critique | Maintenir l'accès pendant les pics de charge |
| API de réservation | Critique | Garantir la création d'inscriptions sans dépassement de jauge |
| Base de données des inscriptions | Critique | Préserver l'intégrité des inscriptions |
| Gestion des jauges | Critique | Empêcher toute surinscription |
| Envoi des billets par e-mail | Élevée | Assurer l'envoi immédiat ou différé des billets |
| Application de scan | Élevée | Permettre le contrôle d'accès même en mode dégradé |
| Back-office | Moyenne | Permettre le pilotage et l'export des données |

---

## 3. Architecture de continuité proposée

L'architecture proposée repose sur une infrastructure redondée et supervisée, dimensionnée pour absorber les pics d'activité au moment de l'ouverture des inscriptions.

```
Utilisateurs
   |
   | HTTPS
   |
CDN / WAF / Protection anti-bot
   |
Load Balancer
   |
+-------------------+      +-------------------+
| API Attendee #1   |      | API Attendee #2   |
| NestJS            |      | NestJS            |
+-------------------+      +-------------------+
          |
          |
+----------------------+   réplication   +----------------------+
| PostgreSQL primaire  | --------------> | PostgreSQL réplica   |
| (instance principale)|   streaming     | (standby / bascule)  |
+----------------------+                 +----------------------+
          |
   Archivage WAL + restauration point-in-time
          |
   Stockage billets / exports
```

Mesures prévues :

- hébergement en Union Européenne ;
- accès HTTPS obligatoire ;
- protection anti-bot et anti-scraping ;
- limitation de débit par adresse IP, appareil et adresse e-mail ;
- plusieurs instances applicatives derrière un load balancer ;
- base de données en haute disponibilité : une instance primaire et un réplica synchronisé en continu (réplication streaming) ;
- archivage continu des journaux de transactions et supervision de la base ;
- monitoring applicatif et infrastructure ;
- astreinte technique pendant les jours d'événement ;
- procédures de reprise documentées.

---

## 4. Objectifs de reprise

| Incident | RTO cible | RPO cible |
|---|---|---|
| Indisponibilité d'une instance API | 5 minutes | 0 à 1 minute |
| Saturation temporaire de l'API | 10 à 15 minutes | 0 à 1 minute |
| Incident base de données avec bascule ou restauration | 5 minutes | Proche de 0 |
| Incident fournisseur e-mail | Service maintenu, envoi différé | 0 minute |
| Coupure Internet locale pour le scan | Continuité en mode hors-ligne | Synchronisation différée |
| Incident back-office | 5 minutes | Proche de 0 |

Le RTO correspond au délai maximal de reprise du service.
Le RPO correspond à la perte maximale de données acceptable.

---

## 5. Gestion des pics d'inscription

Les ouvertures de créneaux peuvent provoquer des pics soudains de trafic. Pour limiter le risque de saturation, les mesures suivantes sont prévues :

### Avant l'ouverture des inscriptions

- test de charge préalable sur les parcours critiques ;
- dimensionnement de l'infrastructure selon le volume attendu ;
- activation du monitoring renforcé ;
- vérification des sauvegardes ;
- vérification des jauges et règles d'inscription ;
- vérification des protections anti-bot ;
- présence d'un référent technique en astreinte.

### Pendant l'ouverture des inscriptions

- surveillance temps réel des temps de réponse ;
- surveillance des erreurs API ;
- surveillance de la charge base de données ;
- surveillance du nombre d'inscriptions par créneau ;
- activation possible d'une file d'attente virtuelle ou d'un mécanisme de régulation de trafic ;
- priorisation des endpoints critiques : consultation des créneaux, réservation, confirmation.

### En cas de saturation

La procédure de continuité est la suivante :

1. détection automatique via monitoring ;
2. analyse du point de saturation : API, base de données, réseau, protection anti-bot ou fournisseur e-mail ;
3. activation du mode de régulation du trafic si nécessaire ;
4. augmentation des ressources applicatives si nécessaire ;
5. suspension temporaire des traitements non critiques ;
6. maintien prioritaire des réservations et de la cohérence des jauges ;
7. information du client sur l'état de la plateforme ;
8. retour à la normale après stabilisation.

---

## 6. Garantie d'intégrité des jauges

La gestion des jauges est considérée comme un service critique.

Lorsqu'un utilisateur s'inscrit à une activité, la réservation doit être validée côté serveur afin d'éviter toute surinscription.

Principes retenus :

- contrôle serveur systématique de la capacité restante ;
- fermeture automatique d'un créneau dès que sa jauge est atteinte ;
- impossibilité de créer une inscription si la capacité est atteinte ;
- gestion transactionnelle des inscriptions concurrentes ;
- journalisation des inscriptions et tentatives refusées ;
- traçabilité des actions d'ouverture et de fermeture des créneaux.

En cas d'incident majeur sur la plateforme, les inscriptions en ligne peuvent être temporairement suspendues afin de préserver l'intégrité des jauges. Aucune réservation hors-ligne n'est effectuée sur les créneaux à capacité limitée, afin d'éviter les conflits et les dépassements de capacité.

---

## 7. Mode dégradé pour les inscriptions

En cas d'incident empêchant temporairement l'inscription en ligne, les mesures suivantes sont prévues :

- affichage d'un message d'indisponibilité temporaire ;
- maintien d'une page d'information accessible ;
- conservation des créneaux et jauges déjà enregistrés ;
- reprise des inscriptions après rétablissement du service ;
- possibilité pour l'organisateur de prolonger une période d'inscription ou de rouvrir un créneau depuis le back-office ;
- communication coordonnée avec le client en cas d'interruption prolongée.

Le mode dégradé ne permet pas de créer des inscriptions hors-ligne lorsque des jauges strictes sont appliquées. Cette limite est volontaire afin de garantir l'équité d'accès et d'éviter les sur-réservations.

---

## 8. Continuité du contrôle d'accès

L'application de scan est conçue pour permettre un fonctionnement en mode dégradé.

Avant le début des contrôles, les listes d'inscrits et les billets valides sont synchronisés sur les terminaux autorisés.

En cas de coupure réseau locale :

- les agents peuvent continuer à scanner les QR codes déjà synchronisés ;
- l'application affiche un retour visuel immédiat : valide, invalide ou déjà scanné localement ;
- les scans sont enregistrés localement ;
- les données sont synchronisées automatiquement lors du rétablissement de la connexion.

En cas de scan simultané du même billet sur plusieurs appareils en mode hors-ligne, la réconciliation est effectuée lors de la synchronisation. Le premier scan valide est conservé, les éventuels doublons sont signalés dans le back-office.

Mesures complémentaires :

- affectation des terminaux par salle ou par entrée ;
- synchronisation préalable avant chaque créneau ;
- supervision du taux de remplissage par espace et par créneau ;
- export post-événement des présences.

---

## 9. Gestion d'un incident fournisseur e-mail

L'e-mail de confirmation et le billet numérique sont importants, mais ne doivent pas bloquer la création de l'inscription.

En cas d'incident sur le fournisseur d'e-mail :

- l'inscription reste enregistrée ;
- le billet reste généré ou disponible à la génération ;
- l'envoi de l'e-mail est placé en file d'attente ;
- les envois sont rejoués automatiquement après rétablissement ;
- un renvoi manuel peut être effectué depuis le back-office.

Ainsi, l'indisponibilité temporaire du service e-mail n'entraîne pas de perte d'inscription.

---

## 10. Sauvegardes et restauration

Les données d'inscription, de présence et de paramétrage sont protégées par une base de données en haute disponibilité, doublée d'un archivage continu.

La stratégie repose sur deux mécanismes complémentaires :

- **Réplication streaming (haute disponibilité)** : une instance PostgreSQL primaire et un réplica reçoivent les transactions en flux continu. En cas de défaillance de l'instance primaire, le service bascule sur le réplica, qui contient une copie quasi temps réel des données. La perte de données est réduite à quelques secondes au maximum (RPO proche de 0).
- **Archivage WAL + restauration point-in-time (PITR)** : en complément de la réplication, les journaux de transactions sont archivés en continu et une sauvegarde de base est réalisée périodiquement. Ce second filet permet de reconstruire la base à n'importe quel instant, même en cas de corruption logique répliquée sur les deux instances.

Mesures prévues :

- instance primaire et réplica synchronisés en continu par réplication streaming ;
- bascule sur le réplica en cas de défaillance de l'instance primaire ;
- archivage continu des journaux de transactions (WAL) avec sauvegarde de base périodique ;
- restauration point-in-time (PITR) en cas de corruption logique ;
- conservation des sauvegardes pendant une durée définie avec le client ;
- vérification périodique de la capacité de restauration et de la santé du réplica ;
- export CSV disponible depuis le back-office ;
- export de sécurité avant les ouvertures majeures et avant l'événement.

En cas de défaillance de l'instance primaire, la procédure de reprise consiste à basculer le service sur le réplica, puis à vérifier l'intégrité des inscriptions et des jauges. En cas de corruption logique des données, la reprise s'appuie sur l'archivage WAL et la restauration point-in-time à partir de la dernière sauvegarde de base.

---

## 11. Monitoring et alerting

La plateforme est supervisée pendant toute la période critique.

Indicateurs surveillés :

- disponibilité de la page publique ;
- disponibilité de l'API ;
- taux d'erreurs HTTP ;
- temps de réponse API ;
- charge CPU et mémoire ;
- connexions base de données ;
- latence base de données ;
- volume d'inscriptions par minute ;
- taux de remplissage des créneaux ;
- échecs d'envoi e-mail ;
- erreurs applicatives.

Alertes prévues :

- alerte en cas d'indisponibilité ;
- alerte en cas de hausse du taux d'erreurs ;
- alerte en cas de saturation de la base de données ;
- alerte en cas de file d'attente e-mail anormale ;
- alerte en cas de comportement suspect ou automatisé.

---

## 12. Astreinte technique

Une astreinte technique est prévue pendant les deux jours de l'événement et pendant les ouvertures critiques d'inscriptions.

L'astreinte couvre :

- surveillance active de la plateforme ;
- analyse des incidents ;
- coordination avec l'organisateur ;
- correction ou contournement des incidents ;
- relance des services si nécessaire ;
- communication de statut.

Organisation proposée :

| Niveau | Rôle | Mission |
|---|---|---|
| Niveau 1 | Référent projet | Communication avec le client et qualification de l'incident |
| Niveau 2 | Référent technique | Diagnostic applicatif et infrastructure |
| Niveau 3 | Hébergeur / infrastructure | Intervention sur ressources serveur, réseau ou base de données |

---

## 13. Procédure d'incident

### Niveau 1 — Incident mineur

Exemples :

- ralentissement ponctuel ;
- e-mail retardé ;
- erreur isolée sur un utilisateur.

Actions :

1. analyse des logs ;
2. correction ou contournement ;
3. suivi jusqu'au retour à la normale.

### Niveau 2 — Incident majeur partiel

Exemples :

- ralentissement important à l'ouverture d'un créneau ;
- hausse du taux d'erreurs ;
- difficulté d'accès temporaire au back-office.

Actions :

1. activation de l'astreinte ;
2. analyse du composant concerné ;
3. régulation du trafic si nécessaire ;
4. augmentation des ressources si nécessaire ;
5. communication au client ;
6. suivi jusqu'au retour à la normale.

### Niveau 3 — Incident critique

Exemples :

- indisponibilité de la billetterie ;
- indisponibilité de la base de données ;
- impossibilité de créer des inscriptions.

Actions :

1. activation immédiate du PCA ;
2. gel temporaire des inscriptions si nécessaire ;
3. bascule ou restauration du service ;
4. vérification de l'intégrité des données ;
5. réouverture contrôlée des inscriptions ;
6. communication client toutes les 15 minutes jusqu'au rétablissement.

---

## 14. Tests préalables

Avant la mise en production, les contrôles suivants sont prévus :

- recette fonctionnelle complète du parcours d'inscription ;
- test de création d'inscription ;
- test de fermeture automatique des jauges ;
- test de limitation par appareil, adresse IP ou adresse e-mail ;
- test d'émission du billet numérique ;
- test de scan valide, invalide et déjà utilisé ;
- test d'export CSV ;
- test de charge sur les endpoints critiques ;
- test de restauration ou vérification des sauvegardes ;
- test de l'astreinte et des procédures d'escalade.

Un test complet de bout en bout est réalisé avant l'ouverture des inscriptions.

---

## 15. Limites du PCA

Le PCA garantit la continuité et la reprise des services dans les conditions prévues, mais certains cas peuvent nécessiter une interruption temporaire contrôlée.

Limites identifiées :

- aucune inscription hors-ligne n'est réalisée sur les créneaux à jauge limitée ;
- en cas d'indisponibilité totale de la plateforme, les nouvelles inscriptions sont suspendues jusqu'au rétablissement ;
- l'envoi d'e-mail peut être différé sans perte d'inscription ;
- le mode hors-ligne concerne le contrôle d'accès, pas la réservation de places ;
- les délais de reprise dépendent du niveau d'infrastructure retenu et des engagements de l'hébergeur.

---

## 16. Livrables associés

Les livrables suivants peuvent être fournis au client :

- plan de continuité d'activité ;
- procédure de gestion d'incident ;
- rapport de test de charge ;
- guide utilisateur back-office ;
- guide de scan pour les agents ;
- politique de sauvegarde et restauration ;
- export de sécurité avant événement ;
- rapport post-événement avec statistiques d'inscription et de présence.

---

## 17. Synthèse

La continuité de service repose sur trois principes :

1. maintenir la plateforme disponible pendant les pics d'inscription ;
2. préserver l'intégrité des jauges en toutes circonstances ;
3. permettre le contrôle d'accès en mode dégradé lorsque le réseau local est indisponible.

En cas d'incident sur la billetterie, la priorité est donnée à la protection des données, à la cohérence des inscriptions et à une reprise rapide du service. En cas d'incident réseau sur site, le contrôle d'accès peut continuer en mode hors-ligne avec synchronisation différée.
