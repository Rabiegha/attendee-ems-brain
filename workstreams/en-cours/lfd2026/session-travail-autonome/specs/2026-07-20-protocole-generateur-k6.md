# Protocole k6 — atteindre la limite d'Attendee, pas celle du générateur

> **Date :** 2026-07-20
>
> **Cible autorisée :** staging uniquement
>
> **But :** distinguer sans ambiguïté une saturation d'Attendee d'une saturation de la machine k6

## 1. Principe

Un résultat de charge n'est valable que si k6 a réellement injecté la charge annoncée. Configurer
`3000 VUs` ne prouve ni que 3 000 requêtes ont démarré ensemble, ni que le générateur a tenu le débit,
ni que 3 000 inscriptions métier uniques ont été traitées.

La campagne sépare donc trois questions :

1. **débit stable** : combien de nouvelles opérations par seconde Attendee absorbe durablement ;
2. **burst de 3 000** : en combien de temps 3 000 inscriptions uniques sont absorbées ;
3. **choc de concurrence** : que se passe-t-il lorsque 3 000 utilisateurs déclenchent une opération
   dans une fenêtre extrêmement courte.

La vraie limite est le dernier palier stable et reproductible avant que le débit utile plafonne tandis
que les latences, les erreurs ou les files augmentent. Un pic isolé non reproductible n'est pas une
capacité démontrée.

## 2. Deux modèles k6 distincts

### Débit contrôlé — modèle ouvert

Utiliser `ramping-arrival-rate` ou `constant-arrival-rate` pour imposer un nombre d'itérations par
seconde indépendamment du temps de réponse d'Attendee.

- aucun `sleep()` dans une itération pilotée par arrival rate ;
- VUs préalloués avant le palier ;
- `dropped_iterations` surveillé et inclus dans le verdict ;
- paliers progressifs, par exemple 25, 50, 100, 200 puis 300 opérations/s ;
- chaque palier dure assez longtemps pour observer DB, pool, CPU, mémoire et récupération.

Ce modèle trouve le débit stable et le point d'inflexion de l'architecture.

### Choc de concurrence — modèle fermé one-shot

Pour « 3 000 personnes, une inscription chacune » :

- préparer exactement 3 000 identités uniques ;
- utiliser une itération unique par utilisateur, sans boucle pendant 60 secondes ;
- préinitialiser tous les VUs avant le déclenchement ;
- synchroniser les générateurs ;
- mesurer la distribution réelle des timestamps de départ ;
- vérifier après test exactement 3 000 résultats métier, répartis selon capacité/waitlist.

Ce scénario mesure un choc. Il ne signifie pas que le système soutient 3 000 requêtes par seconde
pendant plusieurs minutes.

## 3. Dimensionnement initial du générateur

Avant de viser 3 000 VUs :

1. exécuter le script final avec 100 VUs sans sortie verbeuse ;
2. mesurer CPU, RAM physique, réseau, sockets et durée d'initialisation ;
3. estimer la mémoire cible à partir de la consommation par VU, avec marge ;
4. recommencer à 250 puis 500 VUs ;
5. répartir la charge dès que la marge du générateur devient insuffisante.

Critères de validité du générateur pendant un test :

- CPU k6/générateur inférieur à 80 % ;
- RAM physique inférieure à 90 % et aucun swap actif ;
- bande passante non saturée ;
- aucune erreur `too many open files`, port éphémère ou socket locale ;
- débit réellement démarré conforme au débit demandé ;
- `dropped_iterations = 0` pour déclarer le scénario complètement injecté.

Un `dropped_iterations` tardif peut aussi révéler qu'Attendee ralentit tellement que tous les VUs
restent occupés. Il doit donc être corrélé aux ressources k6 et serveur, pas interprété seul.

## 4. Optimisation du script k6

- charger les comptes/identités avec `SharedArray`, avec `open()` à l'intérieur de son callback ;
- mettre `discardResponseBodies: true` par défaut et ne conserver le body que pour les requêtes dont
  le contrat doit réellement être vérifié ;
- générer les payloads simples en amont plutôt que faire un travail JS coûteux dans chaque itération ;
- supprimer tout `console.log` par requête ;
- limiter checks, groupes et métriques personnalisées aux mesures utiles ;
- donner un nom stable aux URLs pour éviter une série métrique par identifiant/token ;
- garder les dépendances JS minimales ;
- ne pas écrire un énorme flux JSON local pendant le test maximal ;
- exporter les métriques vers le système central prévu et produire le résumé après le test ;
- rendre les checks robustes aux réponses sans body, aux timeouts et aux erreurs réseau.

Le script de capacité doit rester plus léger que le système qu'il mesure.

## 5. Répartition sur plusieurs générateurs

Pour le scénario maximal, utiliser plusieurs machines externes à l'infrastructure Attendee. Point de
départ recommandé : quatre générateurs, chacun responsable d'un segment de 750 utilisateurs pour le
choc de 3 000.

- machines de taille et configuration identiques ;
- horloges synchronisées ;
- même version k6, même script et même SHA ;
- données partitionnées sans email/token partagé ;
- `execution-segment` pour éviter le double envoi d'itérations ;
- départ coordonné via k6 Cloud, k6 Operator ou mécanisme pause/reprise contrôlé ;
- métriques centralisées afin d'agréger les résultats des quatre instances.

En exécution OSS segmentée, les seuils sont évalués séparément par instance. Le verdict global doit
donc être calculé sur les métriques agrégées et les contrôles métier serveur.

## 6. Comment identifier le vrai goulot

| Observation | Conclusion probable |
| --- | --- |
| CPU/RAM/réseau k6 saturé, serveur encore calme | générateur limité ; test invalide pour Attendee |
| `dropped_iterations` dès le début, serveur calme | VUs préalloués insuffisants ou générateur trop petit |
| `dropped_iterations` après hausse forte des latences serveur | Attendee ralentit ou `maxVUs` insuffisant ; corréler |
| débit injecté augmente mais débit utile plafonne, serveur/DB saturé | limite Attendee atteinte |
| doubler les générateurs augmente encore le débit | ancien générateur était probablement le plafond |
| doubler les générateurs ne change plus le débit et le serveur sature | plafond Attendee probablement confirmé |
| p95/p99 montent sans ressource serveur saturée | chercher verrous, pool, réseau, dépendance externe |

Le palier limite doit être répété au moins deux fois. Pour le confirmer, refaire le même scénario avec
davantage de capacité côté générateur. La conclusion n'est solide que si le plafond reste côté
Attendee et non côté client de charge.

## 7. Ordre de campagne

1. valider les invariants avec faible charge ;
2. calibrer un générateur à 100, 250 et 500 VUs ;
3. mesurer 25 → 50 → 100 → 200 → 300 opérations/s en arrival rate ;
4. ajouter des paliers intermédiaires autour du premier point d'inflexion ;
5. répéter le meilleur palier stable ;
6. distribuer le test si un générateur approche ses limites ;
7. exécuter 3 000 inscriptions uniques sur 60 s, puis 30 s, puis 10 s ;
8. seulement ensuite exécuter le choc one-shot de 3 000 utilisateurs ;
9. mesurer le retour à la normale et contrôler les données en base ;
10. produire les rapports par test et le bilan LFD.

Tout palier est interrompu si une violation d'intégrité apparaît. La recherche du plafond ne justifie
jamais de continuer à produire du surbooking ou de la corruption.

## 8. Sources de référence

- [Grafana k6 — Running large tests](https://grafana.com/docs/k6/latest/testing-guides/running-large-tests/)
- [Grafana k6 — Constant arrival rate](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/)
- [Grafana k6 — Dropped iterations](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/dropped-iterations/)
- [Grafana k6 — SharedArray](https://grafana.com/docs/k6/latest/javascript-api/k6-data/sharedarray/)
- [Grafana k6 — Running distributed tests](https://grafana.com/docs/k6/latest/testing-guides/running-distributed-tests/)
