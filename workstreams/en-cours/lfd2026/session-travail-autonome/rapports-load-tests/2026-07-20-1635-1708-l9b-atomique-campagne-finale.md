# Campagne k6 LFD — L9.b atomique et limite mesurée

## Verdict exécutif

- Environnement : **staging uniquement**, production surveillée en lecture seule.
- Correction applicative : PR backend #50, réservation de la place et création du choix dans une
  instruction PostgreSQL atomique.
- SHA principal testé : `4bd938d55f11576d1a25291952fe725f636d535c`, puis
  `f37df976dcbb18b1e8a67d5de92e9ed990ff8f71` après ajout du client Gotenberg B0 désactivé et non
  branché au flux d'inscription.
- **Débit soutenu prouvé : 70 inscriptions/s pendant 2 minutes.**
- **Choc simultané prouvé : 250 inscriptions**, démarrées dans une fenêtre de 10 ms.
- 75/s soutenues, 350 simultanées et 500 simultanées ne sont pas validées.
- Le besoin contractuel de 3 000 simultanées **n'est pas prouvé** par cette campagne mono-générateur
  et ne doit pas être extrapolé à partir de 250.

## Résultats soutenus

| Palier | Durée utile | Confirmations | Débit | p95 | p99 | Max | Erreurs/pertes | Verdict |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| smoke | 1 requête | 1/1 | — | 157 ms | 157 ms | 157 ms | 0 | vert |
| 25/s | 120 s | 3 001/3 001 | 24,99/s | 113,96 ms | 156,45 ms | 296,51 ms | 0 | vert |
| 50/s | 120 s | 6 001/6 001 | 49,99/s | 164,68 ms | 237,76 ms | 392,79 ms | 0 | vert |
| 60/s | 120 s | 7 201/7 201 | 59,97/s | 149,62 ms | 247,70 ms | 438,57 ms | 0 | vert |
| 65/s | 120 s | 7 801/7 801 | 64,98/s | 205,88 ms | 530,91 ms | 1,84 s | 0 | vert avec réconciliation manuelle |
| 70/s | 120 s | 8 400/8 400 | 69,97/s | 222,98 ms | 405,37 ms | 1,00 s | 0 | vert |
| 75/s | arrêt vers 4 s | 107 reçues | 26,89/s | 2,39 s | 2,67 s | 3,10 s | 49 dropped | rouge |

Le palier 75/s a créé 248 inscriptions en base après l'arrêt pour seulement 107 confirmations reçues.
Il illustre le risque UX d'un client interrompu pendant que les transactions déjà engagées finissent
par committer. Les compteurs sont restés exacts et aucune capacité n'a été dépassée.

## Résultats des chocs simultanés

| Choc | Dispersion de départ | Réponses confirmed | p95 | p99 | Max | Résultat DB après cooldown | Verdict |
| --- | ---: | ---: | ---: | ---: | ---: | --- | --- |
| 100 | 4 ms | 100/100 | 2,15 s | 2,28 s | 2,37 s | 100 exactes | vert |
| 250 | 10 ms | 250/250 | 497,73 ms | 637,49 ms | 792,34 ms | 250 exactes | vert |
| 350 | 11 ms | 321 reçues, 2 timeouts | 1,02 s* | 1,36 s* | 1,64 s* | 327, dont 6 sans confirmation reçue | rouge |
| 500 | 15 ms | 448 reçues, 12 timeouts | 3,90 s* | 5,08 s* | 5,54 s* | 448 exactes | rouge |

`*` Les percentiles des runs interrompus ne portent que sur les réponses observées avant l'arrêt ; ils
ne résument pas les utilisateurs encore bloqués ou interrompus et ne doivent pas être lus comme un
succès UX.

Le fait que le choc 250 ait une latence meilleure que le choc 100 montre aussi la variabilité d'un
échantillon unique (cache, checkpoint, ordonnancement). La conclusion robuste est le succès ou
l'échec complet avec invariants, pas une comparaison fine entre ces deux percentiles isolés.

## Invariants métier et audit

- Tous les runs terminés conservent `Session.registered_count` égal au nombre réel de choix
  confirmed et ne dépassent jamais la capacité.
- Les runs verts ont exactement un participant, un email, une inscription, un choix confirmed et un
  audit HTTP 2xx par succès.
- Le run 350 révèle six commits sans réponse reçue : 327 inscriptions/choix en DB contre 321 audits
  2xx observés. Cela confirme qu'idempotence et réconciliation UX restent indispensables au-delà de
  la zone sûre.
- Aucun restart, OOM ou état incohérent final n'est attribuable à la charge. Un run 65/s distinct a
  été volontairement invalidé lorsque le watchdog a détecté le redéploiement concurrent de B0 ; les
  trois 502 de ce run ne sont pas une limite de capacité.

## Ressources observées

| Palier | Pic CPU API | Pic CPU PostgreSQL | Pic CPU PgBouncer | Mémoire hôte disponible minimale |
| --- | ---: | ---: | ---: | ---: |
| 25/s | 176,47 % | 49,74 % | 17,52 % | 20 083 Mo |
| 50/s | 266,23 % | 81,09 % | 25,44 % | 20 004 Mo |
| 60/s | 275,88 % | 100,90 % | 28,46 % | 19 740 Mo |
| 70/s | 363,88 % | 155,63 % | 29,93 % | 19 734 Mo |

Les pourcentages Docker peuvent dépasser 100 % lorsque plusieurs cœurs sont utilisés. L'hôte dispose
de huit cœurs. Le générateur est resté loin de ses seuils mémoire/CPU ; il n'est pas la limite pour
ces paliers. La santé de la production partagée est restée verte pendant les contrôles read-only.

## Incident de preuve et correction du harnais

Après le run 65/s, la requête finale de réconciliation a dépassé son timeout de 30 s. Elle filtrait
`audit_logs` par `target_id` sans fixer `target_type`, alors que l'index existant est composite
`(target_type, target_id)`. La PR backend #52 ajoute
`target_type = 'session_registration'` : la même vérification est ensuite devenue immédiate et a
confirmé 7 801/7 801 éléments exacts. Le run k6 est valide ; l'échec initial concernait uniquement le
transport de la preuve postflight.

## Avis pour LFD

- **GO technique mesuré** pour 70 inscriptions/s soutenues pendant deux minutes sur cette instance
  et pour un choc de 250 inscriptions réellement quasi simultanées.
- **NO-GO pour annoncer 75/s soutenues, 350 simultanées, 500 simultanées ou 3 000 simultanées.**
- 70/s représente environ 4 200 inscriptions par minute si la demande est lissée ; ce chiffre ne
  remplace pas un test de choc.
- La preuve 3 000 nécessite plusieurs générateurs synchronisés, une barrière commune, une mesure de
  dispersion de départ et une réconciliation globale. Le harnais actuel refuse volontairement plus
  de 500 VUs sur un générateur et refuse le multi-générateur tant que l'orchestrateur n'est pas livré.
- Rejouer au minimum 70/s et 250 simultanées après tout changement qui touche réellement le chemin
  d'inscription, les audits synchrones, les triggers ou le pool DB. Le client Gotenberg B0 isolé ne
  touche pas ce chemin.

## Traçabilité

- PR #50 : `perf(registration): atomically claim session choice`.
- PR #51 : client Gotenberg OIDC B0, isolé et désactivé par défaut.
- PR #52 : utilisation de l'index audit dans la réconciliation k6.
- Artefacts bruts locaux :
  `attendee-ems-back-wt-k6-final/tmp/k6/lfd-l9b-{atomic,b0}-*/gen-01/`.
- Chaque dossier contient métadonnées, configuration de charge, résumé k6, ressources, watchdog,
  sondes live et résultat de réconciliation. Aucun token ni credential n'est inclus dans ce rapport.
