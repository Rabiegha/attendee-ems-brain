# Appli mobile — préparation LFD2026

> Objectif : sortir une **version mobile stable** pour l'event LFD2026.
> Vient de l'inbox `[idea]`. À passer par [NOW.md](../../workspace-rabie/NOW.md) avant implémentation.
>
> **Priorisation** (à valider/trancher) : 🔴 **Event mandatory** (bloquant LFD2026) ·
> 🟡 **Nice to have** · 🔵 **Post-event**. Plusieurs points sont repris/liés au backlog
> technique transverse : [BACKLOG-TECH.md](../a-faire/BACKLOG-TECH.md).

## 🔴 Event mandatory — bloquant pour LFD2026

### Bug éjection mobile — rotation refresh token perdue (HTTP 499)

- Bug confirmé : [bugs/en-cours/2026-07-01-eject-mobile-refresh-rotation-499](../../bugs/en-cours/2026-07-01-eject-mobile-refresh-rotation-499.md)
  (rotation refresh token → 499 → retry avec token révoqué → 401 → logout forcé).
- **Reproduction probable en conditions événementielles** (réseau saturé, app backgroundée
  entre deux scans) → contexte typique LFD2026.
- Fix : (back) grace period sur la rotation (peupler `replacedById`) ; (mobile) retry réseau
  sans jeter le refresh token tant qu'un vrai 401 n'est pas reçu.

### Offline fiable pour les scans en session

- Le mode offline doit fonctionner **sans aucun souci**, surtout pour les scans en session
  (queue offline, resync, **zéro perte de scan**).
- S'appuyer sur les acquis de [mobile-stabilization/03](../../workstreams/fait/mobile-eject-socket-resilient-delta-async/03-socket-resilient-delta-sync.md)
  (socket résilient + delta-sync).

### Couverture des use cases de scan + faux succès / idempotence locale

- Participant **déjà scanné** (double scan), **pas inscrit**, mauvais event/session, QR
  invalide → comportement clair, pas de crash ni d'état ambigu.
- ⚠️ **Faux succès scan session (déjà scanné)** — l'app affiche un **succès** au lieu de
  « déjà scanné » → l'hôtesse ne voit pas le doublon. Fix : détecter le code « déjà
  enregistré » et afficher une confirmation explicite. (cf. [BACKLOG-TECH § urgent](../a-faire/BACKLOG-TECH.md))
- ⚠️ **Faux scans enregistrés en local (idempotence locale manquante)** — les faux succès
  polluent le store local. Fix : ne pas écrire/dupliquer si le serveur répond « déjà
  enregistré » (le serveur est idempotent, pas le store mobile).
- Fichiers : `src/store/registrations.slice.ts`, `src/store/offlineCheckIn.ts`, `src/hooks/useCheckIn.ts`.

### Perf ~20 000 attendees + refonte sync/pagination (⚠️ liés — à traiter ensemble)

- Les deux events LFD cumulent **~20 000 attendees** ; les hôtesses chargent la liste
  complète pour scanner → risque de dégradation perf, surtout en **pic de check-in**.
- À stress-tester : chargement initial (`fetchRegistrationsThunk` + sync par pages de 1000),
  mémoire, fluidité FlatList, coût de reconstruction de l'index Fuse à chaque frappe sur 20k.
- 🔗 **Directement lié à** : **Sync/pagination mobile fragile** ([BACKLOG-TECH § important](../a-faire/BACKLOG-TECH.md)) —
  offset pagination + dataset live qui mute = trous/doublons (4 patches bricolés). Refonte :
  **pagination par curseur (keyset)** ou **sync delta (`updated_at`)**. → **Workstream dédié
  commun** aux deux : la refonte sync règle à la fois la fragilité ET la perf 20k.
- Fichier : `src/store/registrations.slice.ts`. Pistes perf : index Fuse mémoïsé/persisté,
  virtualisation, recherche serveur au-delà d'un seuil.

### Sync de la liste des scans d'une session en local (manquante)

- Aujourd'hui la liste des **scans d'une session** n'est pas synchronisée en local comme
  celle des attendees → incohérence offline / après reconnexion.
- Fix : même mécanisme de cache/delta-sync que la liste des attendees.

### Bug recherche par entreprise (résultats manquants)

- Certaines entreprises ne remontent aucun résultat alors que des participants existent
  (reproduits : « devoteam »/« devo », « lvmh » sur event « Dîner SoftwareOne / IBM / AWS »).
- **Piste code** : `src/screens/EventDashboard/AttendeesListScreen.tsx` (~L326) — l'index
  Fuse cherche `answers.company` **mais pas `attendee.company`**. Ajouter la clé manquante.

### Sécurité — clé PrintNode en clair dans le repo

- Clé PrintNode en clair (4 profils `eas.json` + `.env`) → **rotate la clé PUIS** déplacer
  dans EAS Secrets. (cf. [BACKLOG-TECH § urgent + sécurité](../a-faire/BACKLOG-TECH.md))

### OTA bloqué en prod

- Stratégie `runtimeVersion` en `{ policy: "fingerprint" }` sans `@expo/fingerprint` installé
  → build EAS échouait ; mitigation = version statique `"1.1.9"`. **Reste** : trancher UNE
  stratégie définitive + rebuild store depuis HEAD pour que les OTA atteignent enfin les users.
- Fichiers : `app.json`, `eas.json`, `package.json`.

### Faux positif impression COMPLETED (print-client, jour-J)

- macOS : `webContents.print` = succès dès le spool CUPS même si rien n'est imprimé.
  Fix : `lpstat` sanity-check avant `PATCH /complete`. App : `attendee-ems-print-client`
  (`main-native.js`). (cf. [BACKLOG-TECH § urgent](../a-faire/BACKLOG-TECH.md))

---

## 🟡 Nice to have

### Bouton « Mot de passe oublié »

- Ajouter un bouton/flow mot de passe oublié sur l'écran de login mobile.

### Pagination des sessions

- Pagination de la liste des sessions d'un event.

### Recherche : email + suggestions « did you mean »

- Intégrer/fiabiliser l'**adresse mail** dans la recherche.
- **Suggestions type « did you mean »** si 0 résultat (ex. « frd » → « Frédéric … ») au
  lieu d'un écran vide.

### Requête export `limit=10000` à chaque ouverture d'EventDetails (petit levier front)

- La page EventDetails déclenche un `allRegistrationsForExport` (`limit: 10000`) à chaque
  chargement, même sans export → gaspillage DB/réseau/RAM (mesuré ~0.3s, **pas** la cause
  des lenteurs, mais **conso inutile**). Fix : lazy / `skipToken` tant que l'export n'est
  pas demandé. Front : `attendee-ems-front` `src/pages/EventDetails/index.tsx`.
  (détail : [BACKLOG-TECH § pas urgent](../a-faire/BACKLOG-TECH.md))

### Bug isRoot → 403 sur /events/:id/stats (back)

- Root bypass le guard mais la couche service l'ignore (role undefined). Fix : `if (user.isRoot) return 'any'`
  en tête des 3 utils de scope. (cf. [BACKLOG-TECH § important](../a-faire/BACKLOG-TECH.md))

---

## 🔵 Post-event

### Refactor check-in / check-out avec historique (à trancher — gros chantier)

- Passer d'un état binaire à un **historique d'actions** check-in/check-out (entrée
  principale ET sessions), check-in → check-out → re-check-in multiples, historique
  consultable, **undo LIFO** (annule la dernière action).
- Back : modèle append-only + endpoints check-in / check-out / undo. Front/mobile : UI
  d'historique + re-check-in (fait réapparaître le bouton check-out).
- ⚖️ Gros morceau back+mobile → **post-event** sauf arbitrage contraire.

### Amélioration expérience connexion — « Se souvenir de moi »

- Option « Se souvenir de moi », redirection vers la dernière page après reconnexion,
  message explicite à l'expiration de session (au lieu d'un 401 muet).

### Données de l'ancien user visibles 1s après logout + re-login

- Après logout user1 / login user2, les données de user1 flashent ~1s. Fix : `persistor.purge()`
  - reset des slices dans le thunk logout, avant navigation. (risque sécurité mineur)

### Flash écran login au démarrage après mise à jour

- ~1s d'écran login avant rehydratation du store. Fix : splash/loader neutre tant que
  l'état auth n'est pas résolu (`_hasHydrated`).

---

## 📌 Points de vigilance (checkpoints avant l'event)

### ✅ S'assurer que c'est résolu : recherche participants lente (14-16s)

- Rafale « search-as-you-type » (1 req/caractère) qui sature le process Node mono-worker →
  la dernière requête attend 14-16s. Mitigation front (debounce 300ms) **déjà faite** ;
  **vérifier** que le point est bien clos côté back (index DB + multi-worker) avant l'event.
  (détail : [BACKLOG-TECH § important](../a-faire/BACKLOG-TECH.md))

### ⚠️ Levier L9 / L9.1 (back) — impact mobile à vérifier

- Si le levier débit L9/L9.1 ([suivi leviers](../../workstreams/en-cours/lfd2026/A-I-leviers/01-suivi-leviers.md))
  change le contrat d'inscription/scan (réponse, statut, timing), **vérifier qu'aucune
  adaptation côté mobile n'est requise** (gestion de la réponse, offline queue, statut live).
