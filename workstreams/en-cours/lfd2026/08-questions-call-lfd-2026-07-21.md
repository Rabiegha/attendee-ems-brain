# Call LFD — Questions à trancher avec le client

> **Date prévue :** mardi 21 juillet 2026
> **Objectif :** obtenir les décisions métier/contractuelles nécessaires pour fermer H, BIL, C2.1,
> J-ENTREES, N-ANTI, O et K sans interpréter seuls le cahier des charges.
> **Conseil :** traiter d'abord la section P0. Pour chaque réponse, noter une décision explicite,
> un owner et une échéance.

## 0. Répartition des quatre points déjà identifiés

| Point | Chantier responsable | État avant le call |
| --- | --- | --- |
| IP : blocage strict à 2 ou signal anti-abus | [N-ANTI](./N-architecture-event-ready/N-ANTI-protection-anti-bot.md) | question client ouverte |
| Maximum deux activités/jour | [H](./H-inscriptions-session/README.md) | règle ajoutée : deux `confirmed` par date `Europe/Paris` |
| Informations obligatoires du `+1` | H/BIL, puis C2.1 | question client bloquante |
| Billet et QR propres du `+1` | B/C2.1/D/H | question client bloquante |
| Rôles et accès back-office | [BIL-RBAC](./BIL-billetterie/README.md#bil-rbac--acces-multi-utilisateurs-demande-par-le-cdc) | matrice proposée, validation client requise |
| Accès/rétention du journal d'audit | [O-ACCESS](./O-audit-lfd/README.md#accès-aux-journaux) | question client ouverte ; pas de viewer MEAE supposé |
| Entrées live et export présent/absent | [J-ENTREES](./J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd) | socle partiel ; définitions et restitution à confirmer |

## P0 — Décisions bloquantes à obtenir

### 1. Fonctionnement du `+1`

Le CDC autorise un `+1` aux deux activités, mais ne définit pas son identité ni son cycle de vie.

Questions à poser :

1. Le `+1` doit-il obligatoirement renseigner **nom et prénom** ?
2. Son adresse email est-elle obligatoire, optionnelle ou interdite pour conserver un formulaire court ?
3. Le même accompagnant suit-il automatiquement le titulaire dans toutes ses activités ?
4. Peut-on choisir un `+1` pour une activité et pas pour l'autre ?
5. Peut-on ajouter, remplacer ou retirer le `+1` après l'inscription ? Jusqu'à quand ?
6. Deux places doivent-elles être réservées ou waitlistées comme un groupe indivisible ?
7. S'il ne reste qu'une place, faut-il refuser le groupe complet ou accepter seulement le titulaire ?
8. Le `+1` compte-t-il dans la limite personnelle des deux activités, ou seulement comme seconde place
   dans chacune des activités du titulaire ?
9. Qui reçoit les communications : titulaire uniquement ou accompagnant également ?

Décision recommandée par Attendee : groupe indivisible, deux places atomiques par activité, même
accompagnant sur les activités choisies, informations minimales prénom/nom, email accompagnant optionnel.

**Décision client :** _à remplir pendant le call_
**Owner implémentation :** _à répartir H/BIL/C2.1_
**Échéance :** _à remplir_

### 2. Billet et QR de l'accompagnant

Le CDC demande des billets nominatifs et des QR uniques, mais ne précise pas le cas du `+1`.

Questions à poser :

1. Le `+1` doit-il recevoir un **billet nominatif distinct** ?
2. Doit-il posséder son **propre QR single-use**, scannable indépendamment du titulaire ?
3. Les deux billets sont-ils envoyés dans le même email au titulaire ou séparément ?
4. Le `+1` peut-il entrer sans le titulaire ?
5. En cas d'annulation du titulaire, le billet du `+1` est-il automatiquement invalidé ?
6. Si l'accompagnant change, faut-il invalider l'ancien QR et générer une nouvelle version ?

Décision recommandée : deux identités et deux QR distincts, réunis éventuellement dans le même email ;
la capacité et l'annulation restent cohérentes pour le groupe.

**Décision client :** _à remplir_
**Owner :** _B/C2.1/D/H à répartir_
**Échéance :** _à remplir_

### 3. Limite de deux activités par jour

Attendee propose la règle suivante : maximum deux sessions au statut `confirmed`, par personne et par
date civile `Europe/Paris`.

À faire confirmer :

1. Confirmez-vous que seuls les choix `confirmed` comptent ?
2. Une session `waitlisted` ne consomme donc pas la limite : est-ce bien attendu ?
3. Une annulation libère-t-elle immédiatement une possibilité d'inscription le même jour ?
4. La limite porte-t-elle sur deux activités même si leurs horaires se chevauchent ?
5. Faut-il en plus interdire explicitement deux activités incompatibles/qui se chevauchent ?
6. La règle s'applique-t-elle également au créneau privé du vendredi matin ?

**Proposition actuelle H-D8 :** `confirmed` uniquement, annulation libératrice, quota remis à zéro à
minuit Europe/Paris.
**Décision client :** _à remplir_

### 4. « Deux inscriptions par appareil/IP et email »

Le texte peut bloquer tous les utilisateurs d'un réseau partagé s'il est appliqué littéralement à l'IP.

Questions à poser :

1. Par « appareil (adresse IP) / adresse mail », attendez-vous deux limites dures indépendantes ?
2. L'objectif réel est-il d'empêcher une personne de réserver plus de deux activités, ou d'empêcher
   une attaque automatisée ?
3. Acceptez-vous que l'email porte la limite métier dure et que l'IP soit seulement un signal anti-abus
   à seuil élevé, compatible avec les réseaux ministériels/universitaires ?
4. Souhaitez-vous un cookie appareil non intrusif en complément, sachant qu'il est supprimable et ne
   constitue pas une identification certaine ?
5. Existe-t-il des réseaux/IP connus qui doivent être allowlistés ou surveillés différemment ?

Décision recommandée : deux activités `confirmed` maximum par email/personne ; IP et cookie comme
signaux N-ANTI, jamais blocage dur à deux par IP.

**Décision client :** _à remplir_
**Owner :** N-ANTI
**Échéance :** avant le k6 bout en bout BIL.

### 5. Rôles et droits du back-office

Le CDC cite administrateur, opérateur et lecteur. Attendee propose trois rôles prédéfinis limités à LFD :
`ADMIN_LFD`, `OPERATOR_LFD`, `READER_LFD`.

Questions à poser :

1. Confirmez-vous ces trois profils, sans besoin d'un quatrième rôle ?
2. Le MEAE doit-il seulement **attribuer un rôle prédéfini**, ou aussi modifier librement chaque
   permission ? Recommandation : attribution uniquement pour le MVP.
3. Qui peut inviter/désactiver un utilisateur et changer son rôle ?
4. Un administrateur LFD peut-il créer un autre administrateur LFD ?
5. Le lecteur peut-il exporter les inscrits, sachant que l'export contient des données personnelles ?
6. L'opérateur peut-il corriger nom/email/statut d'une inscription ?
7. L'opérateur peut-il ouvrir/fermer manuellement les inscriptions ? Recommandation : oui.
8. Peut-il modifier horaires, capacités et contenu ? Recommandation : non par défaut.
9. Peut-il supprimer une inscription ou une session ? Recommandation : non par défaut.
10. Les droits doivent-ils être limités au seul événement LFD ? Recommandation : oui.
11. Quelles actions doivent apparaître dans l'historique/audit consultable par le MEAE ?
12. Qui doit pouvoir consulter cet audit ?
13. Une conservation sécurisée restituable sur demande suffit-elle, ou `ADMIN_LFD` doit-il disposer
    d'un écran et/ou d'un export d'audit limité au seul événement LFD ?
14. Quelle durée de conservation attendez-vous pour ces traces d'accès et d'actions ?

Matrice à montrer pendant le call :
[BIL-RBAC](./BIL-billetterie/README.md#matrice-fonctionnelle-recommandee).

**Décision client :** _à remplir_
**Owner :** Corentin / BIL
**Échéance :** _à remplir_

L'implémentation et la preuve de journalisation sont portées par
[O — audit LFD](./O-audit-lfd/README.md), avec `O-BIL` comme lot d'intégration. Elles ne doivent pas
être comptées comme un sous-chantier BIL supplémentaire.

## P1 — Fonctionnement de la billetterie

### 6. Créneau privé du vendredi matin

1. Un lien secret non indexé suffit-il, ou faut-il contrôler une liste d'invités ?
2. Le lien est-il unique pour tout le créneau ou individuel par invité ?
3. Peut-il être transféré à une autre personne ?
4. Faut-il un code supplémentaire, une expiration ou une révocation ?
5. Quelle date/heure exacte d'ouverture doit être configurée ?
6. Les mêmes règles `+1` et maximum deux activités s'appliquent-elles ?

### 7. Horaires d'ouverture et fermeture

1. Confirmer les ouvertures 09h00, 11h00 et 14h00 pour les plages indiquées dans le CDC.
2. Ces heures sont-elles modifiables par l'administrateur LFD ?
3. La fermeture à jauge pleine doit-elle être automatique et immédiate ? Recommandation : oui.
4. Souhaitez-vous une fermeture manuelle d'urgence par l'opérateur ?
5. Une réouverture est-elle possible après désistement ? Automatique ou manuelle ?
6. Que doit voir l'utilisateur avant ouverture : compte à rebours, heure ou simple état fermé ?

### 8. Waitlist et désistements

1. La waitlist est-elle attendue pour LFD ou faut-il refuser lorsque la jauge est pleine ?
2. Si une place se libère, promotion automatique ou validation manuelle ?
3. Combien de temps l'utilisateur promu dispose-t-il pour confirmer ?
4. Quel email doit être envoyé lors d'une promotion/refus ?
5. Le titulaire et son `+1` doivent-ils toujours être promus ensemble ?

### 9. Email et billet « immédiats »

1. Quel délai maximal acceptez-vous entre confirmation et réception du billet : 30 s, 2 min, 5 min ?
2. La page de confirmation peut-elle afficher « billet en préparation » ?
3. Le téléchargement depuis la page de confirmation est-il requis en plus de l'email ?
4. En cas de retard Mailgun, l'inscription reste-t-elle considérée confirmée ? Recommandation : oui.
5. Faut-il un bouton de renvoi manuel depuis le back-office ?
6. Quelles mentions RGPD exactes doivent figurer dans l'email ?

### 10. Champs du formulaire et identité

1. Confirmer que le titulaire renseigne uniquement nom, prénom et email, plus consentement RGPD.
2. Des champs optionnels sont-ils souhaités ?
3. Quelle règle appliquer aux homonymes ou emails partagés ?
4. L'utilisateur peut-il corriger son email après inscription ?
5. Faut-il vérifier l'email avant de confirmer, malgré l'exigence de parcours très court ?

### 10 bis. Tableau de bord des entrées et export post-event

1. Par « nombre d'entrées », souhaitez-vous le nombre **actuellement dans la salle**, le nombre de
   personnes entrées au moins une fois, ou les deux ? Recommandation : afficher les deux.
2. Confirmez-vous qu'une session Attendee correspond à votre couple salle + créneau, ou faut-il
   agréger plusieurs sessions dans une même salle/plage ?
3. Le taux de remplissage attendu est-il bien `présents maintenant / capacité physique` ?
4. Quel délai de rafraîchissement est acceptable sur le back-office : moins d'une seconde ou 3–5 s ?
5. Dans l'export, `Présent` signifie-t-il « au moins un scan IN admis », même si la personne est
   ensuite ressortie ? Recommandation : oui.
6. L'export doit-il lister seulement les `confirmed`, ou aussi les waitlisted/cancelled/blocked et les
   personnes entrées sans réservation ? Recommandation : tout distinguer, sans appeler ces statuts
   « absents ».
7. Quelles colonnes sont obligatoires en plus de l'email et du statut : nom, session, salle, horaires,
   première entrée, dernière sortie ?
8. Le lecteur LFD peut-il télécharger cet export nominatif, ou seulement voir les compteurs agrégés ?

**Décision recommandée :** dashboard avec présents actuels + visiteurs uniques, remplissage sur la
capacité de check-in, export des confirmés avec `Présent`/`Absent` explicite et lignes distinctes pour
les walk-ins/autres statuts.

**Décision client :** _à remplir pendant le call_
**Owner :** H/L9.1 + J-ENTREES + BIL
**Échéance :** avant répétition terrain J-10

## P1 — Performance, disponibilité et sécurité

### 11. Preuve des 3 000 requêtes simultanées

1. Parlez-vous de 3 000 visiteurs actifs simultanés ou de 3 000 inscriptions envoyées à la même seconde ?
2. Quelle fenêtre d'absorption est acceptable ?
3. Quelle latence utilisateur maximale est acceptable pour confirmer une inscription ?
4. Quel taux d'erreur technique est toléré pendant le pic ?
5. Souhaitez-vous assister à la restitution du test k6 et recevoir le rapport ?

### 12. SLA 99,9 % et maintenance

1. Le 99,9 % est-il un engagement contractuel avec pénalité ou un objectif de service ?
2. Comment la disponibilité est-elle calculée : 24/7 de J-15 à J+1 ou uniquement plages d'ouverture ?
3. Les maintenances annoncées sont-elles exclues du calcul ?
4. Quelle sonde/source de mesure fait foi ? Proposition : sonde externe `/api/health`.
5. Quel délai de prise en charge et de rétablissement attendez-vous ?
6. L'instance unique monitorée avec astreinte/runbooks est-elle acceptée explicitement ?

### 13. CAPTCHA et mode dégradé

1. Turnstile ou équivalent transparent répond-il à votre exigence ?
2. Acceptez-vous un mode fail-open contrôlé si le fournisseur anti-bot est indisponible, avec rate
   limits et règles métier toujours actifs ?
3. Préférez-vous bloquer toutes les inscriptions en cas de panne CAPTCHA ?
4. Qui peut activer le mode `attack` le jour J ?

## P2 — Scan, support et conformité

### 14. Scan et appareils

1. Combien d'appareils et d'agents scanneront simultanément sur une même session ?
2. Qui fournit, prépare et recharge les téléphones/tablettes et combien d'appareils de secours ?
3. Chaque appareil doit-il accéder à toutes les sessions LFD ou recevoir une affectation limitée ?
4. Confirmer qu'aucune impression de badge/billet n'est requise pour LFD.
5. Le `+1` peut-il être scanné séparément et entrer sans le titulaire ?
6. Quel retour exact est attendu pour QR invalide, déjà utilisé, hors session et mode offline ?
7. Souhaitez-vous des sons distincts succès/refus/doublon en plus de la couleur et de la vibration ?
8. Quelle date et quel lieu peut-on bloquer pour la répétition complète, au plus tard J-10, puis
    pour la contre-recette du build final au plus tard J-4 ?

Référence de recette : [M — scan terrain](./M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md).

### 15. Support et astreinte

1. Horaires exacts de présence/support attendus J-15 à J+1 et pendant les deux jours.
2. Canal d'escalade : téléphone, Teams, email ?
3. Contacts décisionnaires pour pause/réouverture/mode attaque ?
4. Temps de réponse garanti attendu ?
5. Formation : nombre de sessions, participants et date ?
6. Livrables attendus : guide back-office, guide scan, PCA, rapport post-event ?

### 16. RGPD et livrables contractuels

1. Qui fournit/signe le DPA et à quelle date ?
2. Quelle durée de conservation après l'événement ?
3. Quel délai de mise à disposition puis suppression des données ?
4. Qui est le DPO/contact pour accès, rectification et effacement ?
5. Confirmer que l'hébergement UE documenté suffit et préciser les justificatifs attendus.
6. Le MEAE valide-t-il les sous-traitants Mailgun, GCP/Gotenberg, OVH et R2 concernés ?
7. Qui reçoit et conserve les exports post-event ?

## Tableau de décisions à remplir après le call

| # | Sujet | Décision | Chantier | Owner | Échéance | Preuve/validation |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Données du `+1` | | H/BIL | | | |
| 2 | Billet/QR du `+1` | | B/C2.1/D/H | | | |
| 3 | Deux `confirmed`/jour | | H | Corentin | | E2E H-T15→H-T19 |
| 4 | IP dure ou signal | | N-ANTI | | | test NAT/k6 |
| 5 | Matrice rôles/accès | | BIL-RBAC | Corentin | | E2E allow/deny |
| 5 bis | Accès/rétention du journal d'audit | | O/O-ACCESS | Rabie | | E2E viewer + politique écrite |
| 6 | Lien privé vendredi | | H/BIL | | | |
| 7 | Waitlist/désistements | | H | Corentin | | |
| 8 | Délai billet/email | | C2.1/B | Rabie | | |
| 8 bis | Entrées live + export présent/absent | | H/J-ENTREES/BIL | Corentin + Rabie | | H-T24→H-T25 + J-E1→J-E6 + répétition K |
| 9 | Critères 3 000 | | N/J/T | | | rapport k6 |
| 10 | SLA/mesure | | K/0-MON | | | export uptime + runbook |
| 11 | CAPTCHA dégradé | | N-ANTI | | | E2E/runbook |
| 12 | Scan/appareils | | M/H/K | | | [recette terrain](./M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md) + PV GO/NO-GO |
| 13 | Support/astreinte | | K | | | planning/runbook |
| 14 | RGPD/DPA | | légal/ops | | | documents signés |

## Sortie attendue du call

Le call est réussi si les cinq décisions P0 sont écrites et si chaque autre question reçoit soit une
réponse, soit un owner et une date de réponse. Ne pas terminer avec « à voir » sans responsable.
