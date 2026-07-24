# Load test — L9.b — débit imposé 50 inscriptions/s

## Verdict rapide

- Statut : **VALIDE comme preuve d'échec du palier**
- Niveau LFD : **ROUGE pour 50/s sur ce SHA**
- Conclusion : le système n'absorbe pas 50 inscriptions/s ; la file transactionnelle croît, les
  seuils sont franchis en six secondes et des inscriptions sont validées après abandon du client.

## Identité, objectif et profil

- Date Europe/Paris : 2026-07-20, 14:03–14:04
- Staging uniquement ; backend `c75febddfa77255766df134db280f2af5f54b781`
- Harnais `f4fe3e9a8f207f976710ca0fc25a8144cfbf4f0a`, k6 2.1.0
- `constant-arrival-rate` : 50/s prévues pendant 2 minutes, 150 VUs préalloués, 400 maximum.
- Seuils : 0 erreur, 0 itération perdue, p95 < 1,5 s, p99 < 3 s.
- Arrêt automatique déclenché à 6 s ; il est incorrect d'extrapoler ce run à 2 minutes.

## Générateur k6

- 174 VUs avaient été provisionnés, 173 ont été interrompus lors de l'arrêt et 24 itérations n'ont
  pas pu démarrer.
- Processus k6 déjà arrêté au premier échantillon du watchdog ; CPU hôte 61,2 %, mémoire libre 66 %,
  swap stable. Le serveur/DB travaillaient encore après l'arrêt.
- Le générateur n'est pas la cause principale : la montée des VUs accompagne la hausse des temps de
  réponse, tandis que la DB continue à committer les requêtes en attente.

## Résultats k6

| Mesure | Résultat | Seuil | Verdict |
| --- | ---: | ---: | --- |
| Réponses confirmed reçues | 102 | 50/s pendant 120 s | ÉCHEC |
| Débit utile observé avant arrêt | 17,09/s | 50/s | ÉCHEC |
| `dropped_iterations` | 24 | 0 | ÉCHEC |
| Erreurs HTTP / 5xx sur réponses reçues | 0 / 0 | 0 | OK partiel |
| Médiane / moyenne | 1,75 / 1,89 s | information | dégradé |
| p95 / p99 | 4,07 / 4,51 s | < 1,5 / 3 s | ÉCHEC |
| Maximum | 4,86 s | information | dégradé |

## Invariants et anomalie UX

- PostgreSQL a finalement validé **275** inscriptions uniques pour ce `RUN_ID`, contre seulement
  **102** confirmations reçues par k6 ; les 173 opérations interrompues ont donc fini par committer.
- Compteur global stocké 7 834 = 7 834 confirmed réels, aucun surbooking, choix manquant, mauvais
  statut ou doublon.
- Audits 2xx observés : 102, car l'audit actuel est déclenché après la réponse applicative.
- Risque UX : un navigateur peut afficher un timeout/échec alors que la place est réellement prise.
  Une relance nécessite idempotence et récupération d'état pour devenir non ambiguë.

## Santé, diagnostic et correction

- Pics échantillonnés : API 61,94 %, PostgreSQL 68,88 %, PgBouncer 17,76 % ; pas de CPU/RAM saturé,
  0 restart et 0 OOM. PostgreSQL atteint 32,49 % de mémoire puis récupère.
- Cause issue de la revue SQL/code : le trigger L9.a mettait à jour la même ligne `events` tôt dans
  chaque transaction, conservant le verrou jusqu'au commit ; le pool de connexions formait une file.
- Correction préparée : trigger de capacité atomique rendu `DEFERRABLE INITIALLY DEFERRED` dans la
  PR back #42. Le garde-fou de capacité reste appliqué au commit.
- Test nécessaire : rejouer exactement 25/s puis 50/s après déploiement du SHA corrigé.

## Avis LFD et traçabilité

- NO-GO pour déclarer 50/s ou 3 000 simultanées sur ce SHA ; intégrité DB néanmoins préservée.
- Artefacts : `attendee-ems-back/tmp/k6/lfd-l9-rate50-20260720/gen-01/`.
