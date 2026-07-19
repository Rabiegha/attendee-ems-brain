# Recette terrain — Scan mobile LFD 2026

> **Propriétaires :** M prépare le build et les scénarios ; K organise le terrain et prononce le GO.
> **Première répétition cible :** au plus tard J-10. **Contre-recette :** au plus tard J-4, sur le
> build destiné à l'événement.

## 1. Préconditions bloquantes

- nombre réel de scanners et appareils confirmés ;
- comptes et permissions scanner LFD préparés, sans compte super-admin partagé ;
- QR de test : valide, déjà utilisé, mauvais event, mauvaise session, non inscrit, session pleine ;
- release candidate installée, version/runtime notés et mise à jour automatique désactivable ;
- session de test avec capacité contrôlée et jeu de données représentatif ;
- accès Wi-Fi, 4G/5G et mode totalement offline disponibles ;
- dashboard serveur, logs et observateurs H/M/K disponibles pendant la répétition ;
- aucun scénario d'impression : hors scope LFD.

## 2. Scénarios de recette

| ID | Scénario | Attendu bloquant |
| --- | --- | --- |
| R1 | QR valide online | une admission, nom/session visibles, son + couleur + vibration de succès |
| R2 | même QR online sur deux appareils en même temps | une admission DB ; second appareil affiche « déjà scanné » avec heure |
| R3 | plusieurs QR différents simultanés, session presque pleine | jamais plus d'admis que la capacité ; refus clair au-delà |
| R4 | QR invalide, autre event, non choisi ou session pleine | aucune admission ; messages et sons de refus distincts/compréhensibles |
| R5 | passage en mode avion puis dix scans valides | dix actions visibles comme en attente, aucune perte au redémarrage prévu par la stratégie retenue |
| R6 | même QR scanné offline sur deux appareils | au replay : une admission serveur, un doublon explicite, aucune ligne locale fantôme |
| R7 | réseau revient de façon instable | replay idempotent, queue finit à zéro, conflits visibles, aucun faux succès |
| R8 | app backgroundée/écran verrouillé puis reprise | caméra, session sélectionnée et authentification restent cohérentes |
| R9 | ouverture de chaque appareil avant coupure réseau | nombre de sessions chargé = nombre attendu ; session recherchable et scannable |
| R10 | fermeture forcée puis redémarrage sans réseau | comportement conforme à la stratégie cold start documentée, sans prétendre être synchronisé si les données manquent |
| R11 | 15–30 min de scan continu au rythme terrain | pas de fuite mémoire visible, surchauffe bloquante, écran figé ou dérive de session |

Le scénario R4 est aussi prouvé automatiquement dans H avec un test concurrent. La répétition vérifie
en plus le téléphone, la caméra, le réseau et la compréhension humaine.

## 3. Preuves à conserver

- date, lieu, build exact et commit des repos mobile/back ;
- appareils, type de réseau et nombre de scanners simultanés ;
- captures/vidéos courtes des quatre feedbacks : succès, refus, doublon, offline ;
- export des scans serveur avant/après et compteur de queue locale ;
- anomalies avec sévérité, reproductibilité, owner, correction et contre-test ;
- décision finale signée M/K : `GO`, `GO avec réserve` ou `NO-GO`.

## 4. Gate GO / NO-GO

`NO-GO` si au moins un des points suivants subsiste : scan perdu, capacité dépassée, doublon présenté
comme nouveau succès, queue non vidée sans explication, mauvais event/session conservé ou
build/typecheck rouge.

Une répétition réussie sur un build antérieur ne valide pas un nouveau build. Toute correction native,
caméra, stockage, authentification ou synchronisation impose une contre-recette ciblée.
