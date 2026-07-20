# Load test — L9.a différé — débit imposé 25 inscriptions/s

## Verdict rapide

- Statut : **VALIDE**
- Niveau LFD : **VERT pour 25 inscriptions/s pendant 2 minutes**
- Conclusion : le palier est absorbé sans erreur, sans perte et avec une latence nettement meilleure
  qu'avant la correction L9.a.

## Identité, objectif et profil

- Date Europe/Paris : 2026-07-20, 14:42–14:44
- Staging uniquement ; backend `fc3851facea63ef407c17529f1fe60e86dbed391`
- Run `lfd-l9-postfix-rate25-20260720`, générateur `gen-01`
- `constant-arrival-rate` : 25/s pendant 120 s ; seuils 0 erreur, 0 perte, p95 < 1,5 s,
  p99 < 3 s.

## Résultats

| Mesure | Résultat | Verdict |
| --- | ---: | --- |
| Réponses confirmed | 3 001 / 3 001 | OK |
| Débit utile | 24,99/s | OK |
| Erreurs / `dropped_iterations` | 0 / 0 | OK |
| Minimum / médiane / moyenne | 44,58 / 86,32 / 91,89 ms | OK |
| p95 / p99 | 131,43 / 193,79 ms | OK |
| Maximum | 335,19 ms | OK |
| DB / choix confirmed / emails distincts | 3 001 / 3 001 / 3 001 | exact |
| `Session.registered_count` global | 11 244 / 11 244 réels | exact |
| Surbooking | non | OK |

## Comparaison et diagnostic

- Avant L9.a différé, le même palier donnait p95 182,41 ms et p99 317,27 ms.
- Après correction : p95 −28 %, p99 −39 %, tout en gardant le compteur L9.a et le compteur L9.b
  synchrones et exacts.
- Pics observés : API 165,39 %, PostgreSQL 48,23 %, PgBouncer 24,76 % ; aucune saturation
  mémoire, 0 restart et 0 OOM.

## Avis LFD et traçabilité

- Preuve solide pour 25 inscriptions/s soutenues sur cette architecture et ce SHA.
- Ne prouve ni 50/s ni 3 000 simultanées.
- Artefacts : `attendee-ems-back/tmp/k6/lfd-l9-postfix-rate25-20260720/gen-01/`.
