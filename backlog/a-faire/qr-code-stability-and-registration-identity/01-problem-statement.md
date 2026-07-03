# 01 — Problem Statement

> **STATUS : TO REVIEW / DISCOVERY**

## 1. Le problème en une phrase

> **Un QR code émis vers un invité ne doit pas dépendre exclusivement d'un identifiant technique qui peut disparaître ou être remplacé suite à une modification, suppression ou réimport de la liste d'inscriptions.**

## 2. Le scénario à risque

Scénario typique observé chez un client SaaS événementiel :

1. L'organisateur **importe** une liste d'invités dans Attendee EMS.
2. Le système **génère les QR codes** associés (badge / email).
3. Les QR codes sont **diffusés** : envoyés par email, imprimés sur des badges, intégrés dans des cartons d'invitation.
4. Quelques jours avant l'événement, l'organisateur souhaite faire une **modification massive** (corriger des entreprises, mettre à jour des types de participant, ajouter/retirer des invités).
5. Pour simplifier, l'équipe support ou l'organisateur **supprime la liste** et la **réimporte** depuis une nouvelle version du fichier.
6. Lors du réimport, les `attendees` ou `registrations` sont **recréés** avec de **nouveaux IDs techniques**.
7. Les anciens QR codes, envoyés ou imprimés avant le réimport, pointent vers des IDs qui n'existent plus en base.
8. Le jour J, l'hôtesse scanne un QR officiel : **le système ne retrouve pas l'invité** ou retrouve un état incohérent.

## 3. Impacts

### Impact invité

- Sentiment d'humiliation à l'entrée (« mon nom n'est pas dans le système »).
- Temps d'attente.
- Perte de confiance dans la marque cliente.

### Impact organisateur (client SaaS)

- File d'attente, congestion à l'accueil.
- Mobilisation imprévue d'hôtesses/superviseurs.
- Remise en cause publique du produit Attendee EMS.
- Demande explicite de garanties contractuelles.

### Impact Attendee EMS (SaaS)

- Tickets support critiques le jour J (P1 opérationnel).
- Coût de gestion humain pendant l'événement.
- Risque de churn et de mauvaise réputation.
- Données incohérentes en base après recovery manuel.
- Difficulté à analyser le vrai responsable (l'utilisateur ? le produit ?).

### Impact technique

- Aucun historique des QR émis (impossible de répondre « ce QR a-t-il été envoyé ? »).
- Aucun moyen propre de révoquer un QR.
- Aucun moyen de résoudre un ancien QR vers une registration recréée.
- Logique de scan fragile, basée sur des identifiants mutables ou supprimables.

## 4. Cause racine probable

Le QR encode aujourd'hui une **donnée technique mutable / supprimable** :

- soit `registration.id` (UUID) ;
- soit `attendee.id` ;
- soit `external_id` lorsqu'il est fourni à l'import.

Voir [03-current-risk-analysis.md](./03-current-risk-analysis.md) pour le détail sourcé dans le code.

Aucun découplage n'existe entre :

- l'**identité physique** de l'invité (la personne réelle) ;
- l'**identité métier** de l'inscription (sa place dans cet événement) ;
- l'**identifiant technique** du record en base ;
- le **credential** que constitue le QR code émis.

## 5. Formulation produit

> Le QR code envoyé à un invité est une **promesse SaaS**.
> Cette promesse doit survivre aux opérations administratives internes (modification, suppression, réimport).
> L'invité ne doit jamais payer le prix d'une opération technique réalisée en interne sur sa donnée.

## 6. Hors-sujet (anti-objectifs)

- Ce backlog ne cherche **pas** à empêcher l'organisateur de modifier sa liste.
- Ce backlog ne cherche **pas** à interdire le réimport.
- Ce backlog ne cherche **pas** à remplacer immédiatement le format du QR code.
- Ce backlog **n'impose pas** une refonte big-bang.

## 7. Ce qu'on cherche réellement

- Une **garantie de continuité** : un QR émis reste valide tant qu'une inscription correspondante existe pour cet invité dans cet événement.
- Une **traçabilité** : on doit pouvoir dire « ce QR a été émis, envoyé, scanné, révoqué, remplacé ».
- Une **résilience au réimport** : la réconciliation d'identité doit retrouver l'invité plutôt que d'en créer un nouveau.
- Un **comportement de scan exploitable** par l'hôtesse, même en cas de divergence (voir [07-scan-behavior.md](./07-scan-behavior.md)).
