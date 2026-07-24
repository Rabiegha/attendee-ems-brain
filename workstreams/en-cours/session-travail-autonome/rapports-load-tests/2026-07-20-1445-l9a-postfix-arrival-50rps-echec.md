# Load test — L9.a différé — débit imposé 50 inscriptions/s

## Verdict rapide

- Statut : **VALIDE comme preuve d'échec du palier**
- Niveau LFD : **ROUGE pour 50/s sur ce SHA**
- Conclusion : le trigger L9.a différé améliore le débit, mais un second verrou précoce sur la ligne
  `sessions` provoque encore une file transactionnelle et des confirmations après abandon client.

## Identité, objectif et profil

- Date Europe/Paris : 2026-07-20, 14:45–14:46
- Staging uniquement ; backend `fc3851facea63ef407c17529f1fe60e86dbed391`
- Run `lfd-l9-postfix-rate50-20260720`, générateur `gen-01`
- `constant-arrival-rate` : 50/s prévues pendant 120 s, arrêt automatique à environ 6 s après
  franchissement des seuils. Il est incorrect d'extrapoler ce run à deux minutes.

## Résultats k6

| Mesure | Résultat | Verdict |
| --- | ---: | --- |
| Réponses confirmed reçues | 121 | insuffisant |
| Débit utile avant arrêt | 20,28/s | ÉCHEC |
| `dropped_iterations` | 14 | ÉCHEC |
| Itérations interrompues | 164 | ÉCHEC |
| Erreurs applicatives parmi les réponses | 0 | OK partiel |
| Minimum / médiane / moyenne | 124,74 ms / 1,58 s / 2,09 s | dégradé |
| p95 / p99 | 4,17 / 4,39 s | ÉCHEC |
| Maximum | 4,45 s | ÉCHEC |

## Invariants et ambiguïté UX

- PostgreSQL a finalement validé **285** inscriptions uniques pour le run, contre **121** réponses
  confirmed reçues : les **164** opérations interrompues ont malgré tout committé.
- DB 285, choix confirmed 285, emails distincts 285, aucun choix manquant et aucun surbooking.
- `Session.registered_count` global 11 529 = 11 529 confirmed réels.
- Le risque UX reste majeur : un timeout affiché comme échec peut correspondre à une place prise.

## Ressources, cause et suite

- Pics observés : API 54,72 %, PostgreSQL 57,80 %, PgBouncer 11,30 % ; la machine n'est pas
  saturée, 0 restart et 0 OOM.
- Par rapport au run précédent à 50/s : 121 réponses contre 102 (+19 %), sans résoudre la file.
- Cause issue de la revue code/SQL : `addChoice` verrouille `sessions` avant plusieurs requêtes et
  conserve ce verrou jusqu'au commit.
- Correction attendue : réservation atomique de la place à la fin de la transaction, avec capacité,
  ouverture, waitlist et fermeture concurrente préservées.

## Avis LFD et traçabilité

- NO-GO pour déclarer 50/s ou 3 000 simultanées sur ce SHA.
- Artefacts : `attendee-ems-back/tmp/k6/lfd-l9-postfix-rate50-20260720/gen-01/`.
