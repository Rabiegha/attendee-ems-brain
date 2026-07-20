# Load test — L9.b — one-shot 100 VUs sans barrière

## Verdict rapide

- Statut : **INVALIDE comme preuve de simultanéité**
- Niveau LFD : **VERT fonctionnel, sans verdict choc**
- Conclusion : 100/100 inscriptions réussissent, mais leurs départs sont étalés par l'initialisation
  des VUs ; « 100 VUs » ne prouve donc pas « 100 requêtes au même instant ».

## Identité et profil

- Date Europe/Paris : 2026-07-20, 14:07–14:08
- Staging, backend `c75febddfa77255766df134db280f2af5f54b781`
- Harnais `f4fe3e9a8f207f976710ca0fc25a8144cfbf4f0a`, k6 2.1.0
- Exécuteur `per-vu-iterations` : 100 VUs, une itération chacun, générateur unique.
- Fixture synthétique isolée, capacité 500 000, aucune waitlist.

## Résultats factuels

| Mesure | Résultat | Verdict |
| --- | ---: | --- |
| Confirmed | 100 / 100 | OK |
| Erreurs / dropped | 0 / 0 | OK |
| Moyenne / médiane | 475,60 / 415,36 ms | information |
| p95 / p99 / max | 1 004,48 / 1 153,48 / 1 234,52 ms | seuils OK |
| Durée totale de lancement/absorption | 4,7 s | pas simultané prouvé |

- DB : 100 identities/emails/choix confirmed, compteur global 7 934 = vérité 7 934, aucun surbooking.
- Le sampler de 10 s n'a pas capturé le burst de moins de 5 s ; ses pics de ressources ne sont pas
  exploitables pour attribuer une limite.

## Correction et avis LFD

- Le scénario a été renforcé avec une barrière epoch commune et la métrique
  `one_shot_request_start_offset_ms` ; un max ≥ 500 ms invalide automatiquement le choc.
- Ce run est conservé pour éviter de transformer a posteriori un succès fonctionnel en fausse preuve
  de concurrence.
- Prochain run : mêmes 100 VUs avec barrière et dispersion versionnée.
- Artefacts : `attendee-ems-back/tmp/k6/lfd-l9-oneshot100-20260720/gen-01/`.
