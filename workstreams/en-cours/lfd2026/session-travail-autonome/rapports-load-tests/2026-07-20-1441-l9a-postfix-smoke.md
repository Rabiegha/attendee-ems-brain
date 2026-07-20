# Load test — L9.a différé — smoke L9.b

## Verdict rapide

- Statut : **VALIDE**
- Niveau LFD : **VERT pour le smoke**
- Conclusion : le parcours d'inscription session fonctionne après le passage du trigger L9.a en
  `DEFERRABLE INITIALLY DEFERRED` ; l'intégrité des deux compteurs reste exacte.

## Identité et profil

- Date Europe/Paris : 2026-07-20, 14:41
- Staging uniquement ; backend `fc3851facea63ef407c17529f1fe60e86dbed391`
- Run `lfd-l9-postfix-smoke-20260720`, générateur `gen-01`
- Une inscription nominale sur la fixture LFD isolée.

## Résultats

| Mesure | Résultat | Verdict |
| --- | ---: | --- |
| Succès | 1 / 1 | OK |
| Durée | 159,93 ms | OK |
| Erreurs / itérations perdues | 0 / 0 | OK |
| DB / choix confirmed / emails distincts | 1 / 1 / 1 | exact |
| `Session.registered_count` global | 8 243 / 8 243 réels | exact |
| Surbooking | non | OK |

## Ressources et traçabilité

- Pics observés : API 0,05 %, PostgreSQL 0,03 %, PgBouncer 0,04 % ; 0 restart, 0 OOM.
- Ce run valide uniquement le fonctionnement nominal, pas un débit.
- Artefacts : `attendee-ems-back/tmp/k6/lfd-l9-postfix-smoke-20260720/gen-01/`.
