# Stratégie de réputation Mailgun jusqu'à LFD 2026

> **Source de vérité unique — dernière mise à jour : 24/07/2026 à 11h30 Paris**
>
> Domaine : `mail.attendee.fr` · Mailgun EU · IP partagée
> Événement : **LFD 2026, les 4 et 5 septembre 2026**
> Palier de réputation à valider avant l'événement : **3 000 emails réels sur une journée**
> Détail historique des lots : [suivi-envois-channelscope.md](./suivi-envois-channelscope.md)

Ce fichier fait foi pour les volumes réellement envoyés, les décisions `GO / HOLD / STOP`, les
critères de délivrabilité et la trajectoire de réputation jusqu'au jour J. Il doit être mis à jour
après chaque lot.

Le périmètre est strictement limité à la réputation d'envoi et à la délivrabilité du domaine.

## 1. Décision actuelle

**Statut au 24/07 à 10h50 Paris : `GO`. Le lot a été suspendu proprement après 61/600 accepted,
puis la règle de nettoyage a été clarifiée : une ancienne livraison réussie reste éligible pour
cette nouvelle édition ; seuls les échecs permanents, plaintes et désinscriptions sont exclus. Après
audit de 5 329 événements Mailgun, 12 adresses supplémentaires en échec permanent ont été retirées.
La reprise de 527 adresses propres a démarré à 10h40 Paris. À sa fin, un pré-nettoyage automatique
sélectionnera jusqu'à 12 adresses propres supplémentaires pour atteindre **600 aujourd'hui**.**

Bilan Mailgun du lot de 600 du 23/07, consolidé le 24/07 au matin sur les 600 `messageId` :

- **600 accepted**, **582 delivered** (97,0 %) ;
- **26 événements permanent failure** (4,33 %), concernant 18 messages jamais livrés et 8 messages
  d'abord livrés puis revenus en échec définitif ;
- parmi ces échecs, 10 adresses étaient déjà en `suppress-bounce` Mailgun ;
- 12 événements temporary failure sur 5 messages ; aucun temporaire encore en attente au bilan ;
- **0 complained**, **0 unsubscribed** ;
- Gmail : 76/76 livrés ; Outlook/Hotmail grand public : 21/21 livrés ; pro/autres :
  477 livrés, 26 permanent failures ;
- aucun signal d'échec SPF, DKIM, DMARC, de réputation ou de blocklist. Un greylisting 451 isolé
  s'est terminé en échec « too old » ;
- verdict : livraison globale verte, mais permanent failures entre 2 % et 5 % : **ORANGE** selon
  les critères §5. Décision : nettoyer et rester au palier de 600, sans monter à 720.

Préparation vérifiée le 23/07 :

- nouvelle newsletter : `/tmp/NL-23-24.html` sur le VPS ;
- nouvelle liste : 1 320 adresses valides, 1 320 uniques, aucune adresse de smoke dans la base ;
- découpe : lignes 1–600 le 23/07, puis lignes 601–1 320, soit 720 adresses, le 24/07 ;
- dry-runs validés à 600 et 720, lien `%unsubscribe_url%` injecté ;
- le script d'envoi de masse exige lui-même un fichier au contenu exact
  `GO <date> <offset>` ; le runner effectue la même vérification avant de l'appeler ;
- smoke initial demandé le 23/07 : un message livré et ouvert sur `rgharghar@choyou.fr`, deux
  messages livrés sur `brais@choyou.fr`. Le second message vers `brais` est un doublon de smoke
  accidentel lors de la reprise d'une session distante encore active ; aucun destinataire de masse
  n'a été touché. Un verrou de processus a été ajouté immédiatement pour empêcher deux exécutions
  réelles simultanées ;
- lot de masse du 23/07 : `GO 2026-07-23 0` reçu ; **600/600 acceptés**, exécution terminée à
  12h59 Paris avec `exit 0` ;
- le 24/07, le smoke de **08h00 Paris** vers `rgharghar@choyou.fr` est
  `accepted → delivered → opened` ;
- le timer de 09h00 s'est arrêté avant tout envoi faute de GO, comme prévu ;
- la liste restante de 720 contenait 7 bounces et 1 désinscription dans les suppressions Mailgun.
  Une liste nettoyée de 712 adresses éligibles a été produite ; les 600 premières, sans doublon avec
  le 23/07, constituent le lot maintenu du jour ;
- `GO 2026-07-24 600` reçu ; lot réduit à **600** lancé manuellement à 10h21 Paris. Le premier
  message est accepté ;
- audit historique déclenché à 10h35 Paris : sur les 720 adresses initialement restantes, 581
  avaient déjà été contactées avant le 23/07 et 139 ne l'avaient jamais été. Le lot a été suspendu
  à 61 accepted, sans verrou résiduel. Parmi les 139 nouvelles, 8 ont été envoyées aujourd'hui et
  une suppression concerne une adresse déjà partie ; **131 adresses nouvelles, non encore envoyées
  et non supprimées restent disponibles** ;
- règle métier clarifiée : les anciennes adresses livrées peuvent recevoir cette nouvelle
  newsletter. Sur les 539 adresses non encore envoyées, l'audit de l'historique Mailgun a trouvé
  12 échecs permanents supplémentaires et aucun complaint/unsubscribe. Ils ont été exclus ; les
  527 adresses restantes ont passé le dry-run et la reprise a démarré à 10h40 Paris ;
- complément du 24/07 : à la fin des 527, le service `warmup-topup-20260724` vérifie que le journal
  contient exactement 588 accepted, relance le nettoyage, puis envoie au maximum 12 adresses propres
  pour atteindre 600 ;
- solde de cette newsletter : smoke programmé le samedi 25/07 à 08h00 Paris, puis nettoyage à chaud
  et envoi du solde à 09h00 Paris. La simulation du 24/07 trouve 98 adresses propres ; le plafond
  technique est fixé à 120 pour envoyer tout le solde recalculé sans forcer un quota ;
- le lundi 27/07 repart sur un **nouveau contenu** et une nouvelle campagne, après le même
  pré-nettoyage obligatoire.

## 2. Ce qui a réellement été fait

### Volumes confirmés

| Date | Lot de masse | Tests/smokes NL | Newsletter acceptée | Autre warm-up documenté |
| ---- | ------------: | ---------------: | ---------------------: | -----------------------: |
| 16/07 | 0 | 0 | 0 | 30 emails internes J1 |
| 17/07 | 100 | 7 | 107 | 0 |
| 18/07 | 250 | 1 | 251 | 0 |
| 19/07 | 0 | 0 | 0 | 0 |
| 20/07 | 350 | 1 | 351 | 0 |
| 21/07 | 600 | 1 | **601** | 0 |
| 22/07 | 0 | 2 | **2** | 0 |
| 23/07 | 600 | 3 | **603** | 0 |
| 24/07 à 10h36 | 61 | 1 | **62** | 0 |
| **Total confirmé** | **1 961** | **16** | **1 977** | **30** |

État canonique :

- **1 977 newsletters acceptées par Mailgun** ;
- **2 007 messages de warm-up documentés** en ajoutant les 30 emails internes J1 ;
- **61 messages du lot du 24/07 sont confirmés avant reprise** ; 527 adresses propres sont en cours
  d'envoi et ne sont pas encore ajoutées au total confirmé ;
- aucun lot de masse le 22/07 : J6a s'est arrêté faute de `GO`, J6b n'a pas été lancé ;
- les lignes 701–750 de l'ancienne base ont été volontairement sautées ;
- le dernier palier de masse réellement validé est donc **600 messages**.

Listes de preuve :

- [CSV — un enregistrement par envoi](./destinataires-nl-warmup-au-2026-07-22.csv) ;
- [TXT — 1 303 adresses uniques](./destinataires-nl-warmup-uniques-au-2026-07-22.txt).

### Hiérarchie des preuves

1. Événements Mailgun filtrés sur l'objet exact et le `messageId`.
2. Journaux d'exécution VPS.
3. CSV quotidien local.
4. Tableau de bord Mailgun global uniquement pour la santé générale du domaine.

Le compteur global `Sent` ne doit jamais être additionné aux lots.

## 3. Ce que l'on chauffe réellement

L'IP Mailgun est **partagée** : elle ne nécessite pas un warm-up d'IP dédié. Le travail porte sur :

- la réputation de `mail.attendee.fr` ;
- la régularité des volumes ;
- la qualité des destinataires et de leurs réactions ;
- la réputation observée séparément chez Gmail, Outlook, Yahoo et les domaines professionnels.

La base Channel Scope est considérée comme **engagée et autorisée**, décision confirmée par le
propriétaire du projet. On ne réouvre pas ce point à chaque palier. En revanche, on retire toujours
les bounces, plaintes et désinscriptions avant chaque nouvel envoi. Seuls des emails réels et attendus
comptent dans cette stratégie de réputation.

## 4. Prérequis permanents de réputation

État constaté au 22/07 :

- SPF présent pour `mail.attendee.fr` ;
- DKIM `s1._domainkey.mail.attendee.fr` présent, clé 1024 bits ;
- DMARC `p=none` présent sur `attendee.fr` et hérité par le sous-domaine ;
- MX Mailgun EU et CNAME de tracking présents ;
- tracking Mailgun et suppressions activés.

Actions :

- [x] mettre en place le pré-nettoyage automatique
      `scripts/ops/prepare-warmup-clean-list.js` +
      `scripts/ops/run-warmup-clean-lot.sh` avant les lots ;
- [ ] demander/générer une clé DKIM **2048 bits** si Mailgun et le DNS le permettent ; 1024 bits
      fonctionne, mais 2048 bits est la cible ;
- [ ] vérifier SPF, DKIM et DMARC sur un email Gmail et un email Outlook reçus ;
- [ ] activer et vérifier Google Postmaster Tools ;
- [ ] conserver une adresse `From` stable par type de flux : newsletter d'un côté, transactionnel de
      l'autre, tout en restant sur `mail.attendee.fr` ;
- [ ] vérifier que les événements `delivered`, `temporary failure`, `permanent failure`, `complained`
      et `unsubscribed` sont consultables et consolidables dans Mailgun ;
- [ ] vérifier les suppressions avant chaque lot.

### Nettoyage obligatoire avant chaque campagne

Avant chaque lot, y compris une reprise ou un complément :

1. normaliser et dédupliquer la liste source ;
2. contrôler la syntaxe et la routabilité DNS de chaque domaine : MX valide ou repli SMTP A/AAAA ;
3. pour toute nouvelle liste, exiger une validation de boîte externe datant de moins de 72 heures.
   Par défaut, seuls les résultats `deliverable` sont autorisés ; `undeliverable`, `risky`, `unknown`
   et toute adresse absente du rapport restent en quarantaine ;
4. exclure les adresses déjà envoyées dans **la campagne courante** pour garantir l'idempotence ;
5. exclure les suppressions Mailgun : bounces, complaints et unsubscribes ;
6. parcourir l'historique Mailgun disponible et exclure tout `failed:permanent`, même si l'adresse
   n'apparaît plus dans la suppression courante ;
7. exclure également les événements `rejected` et alimenter la denylist persistante, protégée en
   lecture/écriture, pour ne pas perdre un échec lorsque les événements Mailgun expirent ;
8. conserver les adresses dont les anciennes campagnes ont été livrées avec succès : un destinataire
   peut recevoir plusieurs contenus légitimes ;
9. ne pas exclure un simple `failed:temporary` s'il a ensuite été livré ;
10. recalculer le volume propre réel, faire un dry-run, puis seulement appliquer le GO humain ;
11. écrire un rapport horodaté avec les volumes, motifs d'exclusion et empreintes SHA-256 des listes.

Le script d'envoi ne doit jamais recevoir directement une liste brute. Le wrapper
`run-warmup-clean-lot.sh` produit d'abord une liste propre atomique, puis transmet uniquement cette
sortie au script d'envoi avec le volume exact recalculé. Tous les appels Mailgun, la pagination et
les contrôles finaux sont **fail-closed** : à la moindre donnée incomplète ou incohérence, aucun
fichier propre n'est publié et aucun envoi ne part.

Le wrapper exige la preuve de validation externe par défaut. La campagne des 23–25/07 constitue une
exception explicitement configurée car sa base avait déjà été nettoyée dans Emailable avant import,
mais l'export de preuve détaillé n'est pas disponible sur le VPS. Le contrôle DNS renforcé a malgré
tout trouvé le 24/07 un domaine supplémentaire non routable et l'a ajouté à la denylist. À partir
du nouveau contenu du 27/07, aucun lot de nouvelles adresses ne part sans export Emailable récent
ou intégration API équivalente.

### Désinscription obligatoire pour les newsletters

Chaque newsletter doit contenir :

- un lien visible dans le corps ;
- `List-Unsubscribe: <https://…>` ;
- `List-Unsubscribe-Post: List-Unsubscribe=One-Click` ;
- un traitement effectif de la désinscription sous 48 heures maximum ;
- l'exclusion des suppressions Mailgun et Attendee avant le lot suivant.

Le one-click unsubscribe concerne les newsletters et messages marketing, pas les confirmations,
billets, mots de passe ou autres emails strictement transactionnels.

## 5. Critères chiffrés GO / HOLD / STOP

Les métriques sont calculées sur le lot concerné et sur le cumul glissant des 7 derniers jours.
Les ouvertures et clics restent informatifs : ils ne peuvent jamais autoriser seuls un palier.

### VERT — GO vers le palier suivant

- plainte : **0 sur le lot** et taux glissant inférieur à **0,1 %** ;
- permanent failures / hard bounces : inférieur à **2 %** ;
- temporary failures : inférieur à **2 %**, sans concentration chez un fournisseur ;
- livraison finale : au moins **95 %** ;
- aucun blocage d'authentification, de réputation ou de blocklist ;
- suppressions et désinscriptions intégrées ;
- Google Postmaster : spam inférieur à **0,1 %** lorsqu'une donnée est disponible.

### ORANGE — HOLD, même volume ou réduction

- une plainte isolée, même si le taux reste inférieur à 0,1 % ;
- hard bounces ou temporaires entre **2 % et 5 %** ;
- livraison entre **90 % et 95 %** ;
- ralentissement ou erreurs concentrées chez Gmail, Outlook, Yahoo ou un domaine professionnel ;
- données trop récentes ou incomplètes.

Action : ne pas augmenter, attendre le bilan final, nettoyer et reprendre au même volume ou plus bas.

### ROUGE — STOP

- taux de plainte supérieur ou égal à **0,1 %** ; le seuil absolu à ne jamais approcher est 0,3 % ;
- hard bounces ou temporaires supérieurs ou égaux à **5 %** ;
- livraison inférieure à **90 %** ;
- blocage provider, erreur SPF/DKIM/DMARC, réputation dégradée ou blocklist ;
- dysfonctionnement du lien de désinscription ou ré-envoi à des suppressions.

Action : arrêter les lots, identifier la cause, nettoyer la liste et obtenir un nouveau GO humain.

## 6. Audit du 22 juillet et rampe réelle jusqu'au 14 août

Les volumes ci-dessous sont des **plafonds**, jamais des quotas à remplir. Un lot ne part que s'il
existe un contenu attendu et assez de destinataires engagés. Aucun destinataire ne reçoit le même
message une seconde fois uniquement pour remplir un palier.

Chaque augmentation est limitée à environ **20 %**. La reprise conserve la cadence du dernier lot
validé, soit environ **1 email toutes les 15 secondes**. La cadence reste stable pendant cette rampe :
ce document cherche à construire la réputation, pas à tester la vitesse maximale.

| Date cible | Plafond réel | Statut | Décision attendue |
| ---------- | -----------: | ------ | ----------------- |
| Mer. 22/07 | **0 masse** | ✅ Terminé | audit complet des 1 312 NL, Postmaster, one-click, suppressions |
| Jeu. 23/07 | **600** | 🟠 Terminé — ORANGE | 600/600 accepted ; permanent failures au-dessus de 2 % |
| Ven. 24/07 | **720** | 🔄 En cours — cible 600 propre | 61 + 527 en cours, puis complément automatique de 12 après nouveau nettoyage |
| Sam. 25/07 | **120 max** | 🕘 Programmé à 09h00 — 98 estimées | smoke à 08h00, nettoyage à chaud, puis tout le solde réel propre de la campagne |
| Lun. 27/07 | **850** | ⏳ À venir | GO après bilan consolidé du 24/07 |
| Mar. 28/07 | **1 000** | ⏳ À venir | GO si VERT ; sinon maintien 720–850 |
| Mer. 29/07 | **1 200** | ⏳ À venir | GO si VERT |
| Jeu. 30/07 | **0 augmentation** | ⏳ À venir | consolidation, nettoyage, rapport par provider |
| Ven. 31/07 | **1 400** | ⏳ À venir | GO si tous les signaux de la semaine sont VERTS |
| Lun. 03/08 | **1 650** | ⏳ À venir | GO après bilan consolidé du 31/07 |
| Mar. 04/08 | **0 augmentation** | ⏳ À venir | consolidation intermédiaire |
| Mer. 05/08 | **1 900** | ⏳ À venir | GO si VERT |
| Jeu. 06/08 | **2 200** | ⏳ À venir | GO si VERT |
| Ven. 07/08 | **0 augmentation** | ⏳ À venir | bilan provider et Postmaster |
| Lun. 10/08 | **2 500** | ⏳ À venir | GO si VERT |
| Mar. 11/08 | **0 augmentation** | ⏳ À venir | bilan provider et Postmaster |
| Mer. 12/08 | **3 000** | ⏳ À venir | premier palier cible, contenu réel et attendu uniquement |
| Jeu. 13/08 | **0 augmentation** | ⏳ À venir | bilan complet du premier 3 000 |
| Ven. 14/08 | **3 000 max** | ⏳ À venir | répétabilité uniquement si le 12/08 est entièrement VERT |

Un jour sans lot ou un HOLD n'est pas rattrapé en doublant le volume du lendemain. Après une pause
de plus de 3 jours, reprendre au dernier palier vert ou un palier en dessous.

## 7. Du 17 août au 3 septembre

### Semaine du 17 au 23 août — stabiliser

- deux ou trois envois réels maximum dans la semaine, entre **2 000 et 3 000** chacun ;
- tous les emails réels envoyés par `mail.attendee.fr` comptent dans le volume quotidien total ;
- ne pas dépasser 3 000 sur une journée sans nouvelle décision ;
- vérifier le lendemain de chaque envoi les bounces, plaintes, désinscriptions et la répartition par
  fournisseur ;
- maintenir le palier au lieu de l'augmenter au moindre signal orange.

### Semaine du 24 au 30 août — confirmer la réputation

- un ou deux envois réels contrôlés proches de **3 000** si tous les signaux sont verts ;
- ne pas augmenter au-delà de 3 000 : l'objectif est la répétabilité, pas un nouveau record ;
- contrôler Google Postmaster Tools et les erreurs Mailgun par fournisseur ;
- confirmer que le taux de plainte reste inférieur à 0,1 % et que les suppressions sont à jour ;
- nommer la personne qui décide `GO / HOLD / STOP` pour la dernière semaine.

### Du 31 août au 3 septembre — protéger la réputation

- aucune campagne artificielle destinée uniquement au warm-up ;
- communications réellement nécessaires seulement ;
- gel des changements DNS, Mailgun, `From`, DKIM et DMARC à partir du **2 septembre au soir** ;
- le 3 septembre : smoke Gmail + Outlook + domaine professionnel, contrôle Mailgun, Postmaster,
  authentification et suppressions ;
- aucune hausse de volume de dernière minute.

## 8. Protection de la réputation les 4 et 5 septembre

Cette section porte uniquement sur les décisions susceptibles de protéger ou dégrader la réputation.

- vérifier avant le premier envoi : Mailgun EU, SPF, DKIM, DMARC, Postmaster et suppressions ;
- envoyer un smoke vers Gmail, Outlook et un domaine professionnel ;
- utiliser les mêmes domaines, adresses `From` et formats déjà validés ;
- compter ensemble tous les messages provenant de `mail.attendee.fr` dans le volume du jour ;
- conserver un débit régulier et éviter les rafales soudaines ;
- surveiller en continu `delivered`, temporaires, permanents, plaintes et erreurs par fournisseur ;
- ne jamais retenter vers une adresse bounced, complained ou unsubscribed ;
- suspendre les communications non indispensables en cas de signal orange ou rouge.

### Circuit breaker de réputation

Passage immédiat en HOLD si :

- temporary failures ≥ 2 % sur 10 minutes ;
- livraison qui tombe sous 95 % sur une fenêtre suffisamment consolidée ;
- erreur d'authentification ou blocage provider ;
- toute plainte, avec STOP si le taux atteint 0,1 % ;
- hard bounces ≥ 2 % sur la vague en cours.

En HOLD : arrêter l'augmentation, réduire ou suspendre les envois non indispensables, analyser les
codes Mailgun par fournisseur et ne reprendre qu'après une décision humaine explicite.

## 9. Règle de mise à jour après chaque lot

Dans les 2 heures, puis le lendemain matin, ajouter ici :

| Champ obligatoire | Valeur |
| ----------------- | ------ |
| Date, objet et audience | — |
| Volume ciblé / accepted | — |
| Delivered et taux | — |
| Permanent failures et taux | — |
| Temporary failures et taux | — |
| Complaints et taux | — |
| Unsubscribes et taux | — |
| Répartition Gmail / Outlook / Yahoo / pro | — |
| Google Postmaster | — |
| Décision `GO / HOLD / STOP` et auteur | — |
| Prochain plafond autorisé | — |

Les événements Mailgun et `messageId` font foi. Les ouvertures/clics servent au diagnostic mais ne
remplacent jamais les métriques de livraison et de réputation.

## 10. Références de décision

- [Mailgun — IP warm-up](https://help.mailgun.com/hc/en-us/articles/1260803448249-Can-you-describe-the-IP-warm-up-process)
- [Mailgun — suppressions](https://help.mailgun.com/hc/en-us/articles/360012287493-Suppressions-Bounces-Complaints-Unsubscribes-Allowlists)
- [Google — règles expéditeurs](https://support.google.com/mail/answer/81126?hl=fr)
- [Yahoo — bonnes pratiques expéditeurs](https://senders.yahooinc.com/best-practices/)

## 11. Décisions maintenues

- domaine principal LFD : `mail.attendee.fr` ;
- pas de deuxième domaine avant LFD 2026 ;
- IP partagée Mailgun conservée ;
- la montée est conditionnée par les signaux, jamais par le calendrier seul ;
- la réputation est travaillée uniquement avec du trafic réel et attendu ;
- aucun double envoi pour rattraper un jour manqué.
