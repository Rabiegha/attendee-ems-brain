# Chantier M — Scan mobile et recette terrain LFD 2026

> **Statut au 20/07 : 🟡 M2 partiel + M3 + M5(typecheck) livrés — corrections scan session et son.**
> M2 : erreurs structurées préservées, déduplication queue, détection ALREADY_SESSION_SCANNED (prêt pour H/L9.1).
> M3 : sons distincts succès/refus/doublon via expo-av (rebuild natif requis).
> M5 : typecheck vert (0 erreur TS). Reste M1 (stabilité réseau), M2 reste (cold start, registrations), M4, M5 build RC, M6 recette terrain.

- **But :** livrer une version mobile stable et prouvée sur les parcours de scan réellement utilisés
  pendant LFD.
- **Owner recommandé :** Rabie côté mobile ; Corentin pour les corrections back rattachées à H/L9.1 ;
  K organise la répétition et le GO opérationnel.
- **Estimation M révisée :** **5,5–9 j-dev**, hors corrections H/L9.1 et hors disponibilité calendaire
  des bénévoles/appareils.
- **Recette obligatoire :** [recette-terrain-scan-lfd.md](./recette-terrain-scan-lfd.md).
- **Source backlog :** [appli-mobile-lfd2026.md](../../../../backlog/en-cours/appli-mobile-lfd2026.md).
- **Impression :** aucune impression n'est prévue pour LFD ; PrintNode/print-client ne fait pas partie
  de la recette scan event.

## Verdict de l'audit statique du 19/07

| Exigence CDC | État prouvé dans le code | Verdict |
| --- | --- | --- |
| Plusieurs scanners sur une session | verrou back par `sessionId:registrationId` et retour idempotent pour le même QR ; aucun E2E concurrent multi-scanner | 🟠 partiel |
| Doubles scans offline | queue persistée/rejouée et serveur idempotent ; pas de signal « déjà scanné », pas de déduplication locale des `session-scan` | 🔴 incomplet |
| Retour sonore et visuel clair | message animé coloré + vibration moyenne/forte ; aucune dépendance ni lecture audio de succès/refus | 🟠 visuel/haptique seulement |
| Toutes les sessions sur chaque appareil | endpoint `GET /events/:eventId/sessions`, liste et historiques préchargés/persistés | 🟠 pas de preuve de complétude terrain |
| Répétition complète avant event | aucun protocole ni PV de recette existant avant cet audit | 🔴 à réaliser |

### Éléments solides à conserver

- verrou synchrone caméra et fenêtre anti-rafale de cinq secondes dans `ScanScreen` ;
- résolution locale stricte des QR connus et validation serveur des QR signés inconnus ;
- queue FIFO persistée, replay à la reconnexion et bannière de conflits pour le check-in event ;
- préchargement des sessions et de leurs historiques ;
- retour visuel distinct succès/erreur/warning et retour haptique ;
- transaction back avec advisory lock pour sérialiser deux scans du **même participant** sur la
  même session.

### Gaps confirmés dans le code

1. **Doublon session invisible.** ~~Le back renvoie le scan existant en HTTP 201. Le mobile affiche donc
   « entrée confirmée » au second scanner au lieu de « déjà scanné à HH:mm ».~~
   **Corrigé M côté mobile (20/07) :** détection `ALREADY_SESSION_SCANNED` + affichage heure dans ScanScreen.
   La correction back (HTTP 409 au lieu de 201) reste dans H/L9.1 ; le mobile est prêt à recevoir le contrat.
2. **Course sur la capacité de check-in.** Les deux `COUNT` de capacité sont exécutés avant la
   transaction et le verrou est par participant, pas par session. Deux participants différents
   scannés simultanément peuvent tous les deux observer une place disponible. → **H/L9.1.**
3. **Replay session insuffisant.** ~~`SessionsService.scanParticipant` transforme l'erreur Axios en
   `Error` simple et perd `status/data`.~~
   **Corrigé (20/07) :** la couche service relaie l'erreur Axios brute ; la mutationQueue classifie
   correctement 409 `ALREADY_SESSION_SCANNED` et réconcilie le store optimiste au replay.
4. **Déduplication locale absente.** ~~Deux `session-scan` offline identiques peuvent rester en queue.~~
   **Corrigé (20/07) :** `enqueue()` rejette le doublon `(sessionId, registrationId, scanType)`.
5. **Cold start offline incomplet.** Les sessions/historiques sont persistés, mais les grandes listes
   de registrations sont volontairement exclues de Redux Persist. Après fermeture forcée de l'app,
   un appareil hors ligne ne dispose donc pas forcément de la liste permettant de résoudre les QR. → **M4.**
6. **Aucun son.** ~~Seuls animation, couleur et haptics sont présents.~~
   **Corrigé (20/07) :** `expo-av` installé, sons WAV générés (`assets/sounds/`), hook `useScanSound`
   câblé dans ScanScreen — son distinct pour succès (880 Hz), erreur (220 Hz) et doublon/warning (440 Hz).
   **Rebuild natif requis** (`npx expo prebuild && eas build`).
7. **Aucun filet automatique mobile.** ~~Aucun test unitaire/E2E mobile ni script `test` n'a été trouvé.
   `npx tsc --noEmit` échoue actuellement sur plusieurs erreurs existantes.~~
   **Typecheck corrigé (20/07) : 0 erreur TS.** Tests E2E mobile → M5/M6.

## Frontières M / H / K / T

- **M** possède l'appareil : UX scan, audio/haptics, queue/replay, cache local, compatibilité OS,
  build/OTA et preuves sur téléphones.
- **H/L9.1** possède le serveur : contrat de doublon session, admission/capacité atomique et E2E
  concurrents multi-scanner.
- **K** possède l'exploitation : planning de répétition, parc d'appareils, réseau, appareils de
  secours, formation, PV et décision GO/NO-GO.
- **T** conserve les E2E API génériques ; les tests métier session restent dans H et les tests
  physiques ne sont jamais remplacés par la CI.

## Lots à livrer

| Lot | Contenu | Estimation M |
| --- | --- | ---: |
| M0 — audit | audit statique, matrice CDC, protocole terrain | 0,5 j — fait |
| M1 — stabilité existante | éjection 499, réseau dégradé, sync/pagination 20k, recherche entreprise, OTA | 2–3 j |
| M2 — correction scan session | conserver erreurs structurées, dédupliquer queue/store, reconciliation et feedback doublon | 1–2 j |
| M3 — feedback scan | sons distincts succès/refus/doublon, légende et mode silencieux explicite | 0,25–0,5 j |
| M4 — données offline | indicateur de synchronisation complète, manifeste sessions/participants, stratégie cold start | 1–1,5 j |
| M5 — release | typecheck vert, build RC, permissions/caméra/background/OTA | 0,25–0,5 j |
| M6 — recette terrain | répétition, corrections courtes, contre-recette et PV signé | 0,5–1 j |
| **Total M** | hors H/L9.1 et disponibilité des testeurs | **5,5–9 j-dev** |

Le correctif serveur de capacité et le contrat de doublon sont estimés dans H/L9.1, pas une seconde
fois dans M.

## Ordre d'exécution

1. fermer avec H le contrat doublon/capacité et ajouter les tests concurrents ;
2. corriger le replay et la déduplication locale de session ;
3. rendre typecheck/build verts et produire la release candidate ;
4. ajouter les sons et clarifier les retours de scan ;
5. tester les scénarios offline/reconnexion sur les appareils de l'événement ;
6. organiser une répétition complète, corriger, puis contre-recetter sur le même build.

## Definition of done event

- succès, refus, doublon et offline possèdent un retour visuel, haptique et sonore non ambigu ;
- N appareils peuvent scanner la même session simultanément sans dépasser la capacité ni créer deux
  admissions pour un même participant ; N est le nombre réel confirmé au call ;
- un double scan sur deux appareils offline produit une seule admission serveur et un avertissement
  explicite lors de la réconciliation ;
- chaque appareil affiche la totalité attendue des sessions et un état « synchronisation terminée » ;
- extinction/redémarrage, mode avion, retour réseau, app en arrière-plan et token expiré sont testés ;
- typecheck, build et scénarios automatisables sont verts ;
- une répétition terrain puis une contre-recette sont consignées dans un PV avec build, appareils,
  résultats, anomalies, owner et décision GO/NO-GO.
