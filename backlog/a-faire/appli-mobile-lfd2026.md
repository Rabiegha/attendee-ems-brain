# Appli mobile — préparation LFD2026

> Objectif : sortir une **version mobile stable** pour l'event LFD2026.
> Vient de l'inbox `[idea]`. À passer par [NOW.md](../../workspace-rabie/NOW.md) avant implémentation.

## Items

### 1. Bouton « Mot de passe oublié »
- Ajouter un bouton/flow mot de passe oublié sur l'écran de login mobile.

### 2. Pagination des sessions
- Mettre en place une pagination au niveau des sessions (liste des sessions d'un event).

### 3. Offline fiable pour les scans en session
- S'assurer que le mode offline fonctionne **sans aucun souci**, surtout pour les scans
  pendant les sessions (queue offline, resync, pas de perte de scans).
- S'appuyer sur les acquis de [mobile-stabilization/03](../../workstreams/en-cours/mobile-stabilization/03-socket-resilient-delta-sync.md)
  (socket résilient + delta-sync).

### 4. Couverture de tous les use cases de scan
- Participant **déjà scanné** (double scan) : comportement clair et visible.
- Participant **pas inscrit** : message explicite, pas de crash ni d'état ambigu.
- Lister et tester tous les autres cas limites (mauvais event, mauvaise session, QR invalide…).

### 5. Investigation globale → version stable pour l'event
- Passer en revue tout ce qui concerne cet event et anticiper les problèmes pour livrer
  une version stable.
- Inclut le bug confirmé d'éjection mobile :
  [2026-07-01-eject-mobile-refresh-rotation-499](../../bugs/a-faire/2026-07-01-eject-mobile-refresh-rotation-499.md)
  (rotation refresh token perdue → 401 → logout) — reproduction probable en conditions
  événementielles (réseau saturé, app backgroundée entre deux scans).

### 6. Perf — chargement d'une liste de ~20 000 attendees (⚠️ problème probable)
- Les deux events LFD cumulent ~20 000 attendees. Les hôtesses vont **charger la liste
  complète pour scanner** → risque de **dégradation des performances**, d'autant qu'on
  prévoit des **pics de check-in**.
- À vérifier / stress-tester : chargement initial (`fetchRegistrationsThunk` + background
  sync par pages de 1000), mémoire, fluidité de la FlatList, coût de reconstruction de
  l'index Fuse à chaque frappe de recherche sur 20k items.
- Piste : index Fuse mémoïsé/persisté, virtualisation, pagination ou recherche côté
  serveur au-delà d'un seuil.

### 7. Sync de la liste des scans d'une session (manquante en local)
- Aujourd'hui la liste des **scans d'une session** n'est pas synchronisée en local comme
  l'est la liste des attendees.
- Il faut mettre à jour la liste des scans de session **en local** (même mécanisme de
  cache/delta-sync que la liste des attendees) pour qu'elle reste cohérente offline et
  après reconnexion.

### 8. Refactor check-in / check-out (entrée principale + sessions) avec historique
- Objectif : passer d'un état binaire à un **historique d'actions** check-in / check-out,
  côté **entrée principale** ET côté **scan de sessions**.
- Doit permettre : check-in → check-out → re-check-in → re-check-out **multiples**, avec
  un historique consultable.
- **Undo = annulation de la dernière action** (LIFO) : on undo le dernier check-in, puis
  le check-out précédent, puis le premier check-in, etc. — approche retenue car c'est la
  plus simple à garder **cohérente** (plutôt que d'annuler une action arbitraire au milieu
  de la chaîne).
- Adapter **back + front/mobile** :
  - Back : modèle d'historique (append-only) + endpoints check-in / check-out / undo.
  - Front/mobile : **UI d'historique** (visualiser la chaîne d'actions), possibilité de
    **re-check-in** (n'existait pas avant), et une fois re-check-in fait → le bouton
    **check-out réapparaît**.
- Bien couvrir tous les use cases (déjà check-in, déjà check-out, undo sur chaîne longue…).

### 9. Bug recherche par entreprise (résultats manquants) — à investiguer
- La barre de recherche mobile permet de chercher par entreprise, mais **certaines
  entreprises ne remontent aucun résultat** alors que des participants existent.
  - Exemples reproduits : event « Dîner SoftwareOne / IBM / AWS » → recherche « devoteam »
    ou « devo » = 0 résultat ; idem « lvmh ».
- **Piste identifiée dans le code** : dans
  `attendee-ems-mobile/src/screens/EventDashboard/AttendeesListScreen.tsx` (~L326), l'index
  Fuse cherche `answers.company` **mais pas `attendee.company`**. Selon l'event, la société
  peut être stockée sur `attendee.company` (champ attendee) plutôt que dans les réponses de
  formulaire → d'où l'absence de match. À confirmer et ajouter la clé manquante.
- Améliorations souhaitées :
  - Intégrer l'**adresse mail** dans la recherche (déjà partiellement présente — à vérifier
    et fiabiliser).
  - **Suggestions type « did you mean »** : si 0 résultat (ex. « frd »), proposer des
    correspondances proches (« Frédéric Ktorza », « Frédéric … ») au lieu d'un écran vide.
