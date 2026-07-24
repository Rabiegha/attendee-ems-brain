# 2026-07-24 — J-ENTREES check-in + dashboard combiné staging

## Cadre de sécurité

- staging servi au SHA back `245427cdf678ba4879ecfa6b0c523c05c2ec18f9` ;
- fixtures créées exclusivement par les API authentifiées ;
- événements `J-RECETTE-*`, participants `@test.invalid` et sessions explicitement isolées ;
- aucun `DELETE`, seeder, reset, truncate ou SQL direct ;
- aucune donnée ou événement staging existant modifié ;
- les fixtures sont volontairement conservées.

Le préparateur historique `prepare-staging-fixture.sh` a été écarté : il crée par SQL, modifie
temporairement le runtime et prévoit un nettoyage destructif, incompatibles avec la consigne de cette
campagne.

## Correction du harnais

`assignedRegistrationId()` utilisait l'identifiant du VU. Chaque boucle d'un VU rescannait donc le
même participant et transformait le baseline en mesure de conflits `409`. La distribution utilise
désormais `exec.scenario.iterationInTest`, ce qui attribue une inscription distincte à chaque
itération globale tant que le pool n'est pas épuisé.

## Résultats

Chaque palier combine le baseline scan avec cinq consoles appelant le snapshot batch toutes les deux
secondes. Seuils : scan p95 `< 500 ms`, p99 `< 1 500 ms` ; dashboard p95 `< 300 ms`, p99 `< 800 ms`.

| Palier | Scans admis | Erreurs | Scan p95/p99 | Dashboard | Verdict |
| --- | ---: | ---: | ---: | ---: | --- |
| 15 VU | 197/197 | 0 | 522/684 ms | actif | 🔴 p95 scan |
| 10 VU | 123/123 | 0 | 603/715 ms | actif | 🔴 p95 scan |
| 5 VU | 67/67 | 0 | **307/397 ms** | 50/50 HTTP 200, p95 **432 ms**, p99 453 ms | 🟠 scans verts, dashboard p95 rouge |

Smoke du palier final : 5/5 checks, scan 136 ms, snapshot 131 ms.

## Invariants après charge

| Événement | `present_count` | `unique_entered_count` | Capacité | Remplissage |
| --- | ---: | ---: | ---: | ---: |
| `J-RECETTE-20260724124912` | 198 | 198 | 300 | 66 % |
| `J-RECETTE-20260724125024` | 124 | 124 | 300 | 41 % |
| `J-RECETTE-20260724125144` | 68 | 68 | 300 | 23 % |

Les trois campagnes convergent exactement : aucune erreur 5xx, aucun doublon compteur et aucune
surcapacité. Le backend est resté sain après les runs.

## Verdict et suite

La cohérence transactionnelle J est prouvée sur ces paliers. Le SLO n'est pas encore fermé :

1. diagnostiquer le p95 du snapshot batch sous cinq pollers (auth/RBAC, requête SQL, réseau) ;
2. rejouer un baseline plus long après correction/diagnostic pour réduire la variance ;
3. exécuter ensuite les paliers supérieurs et le scénario inscription + check-in + dashboard ;
4. ne décider d'un portier Redis qu'après ces mesures — il ne corrigera pas à lui seul la latence du
   dashboard authentifié.

La cible « 3 000 simultanées » reste non prouvée et nécessite un protocole multi-générateur distinct.
