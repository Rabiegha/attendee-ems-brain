# 06 — Learning L9.1 : présence session atomique et O(1)

> Date : 2026-07-20  
> Branche back : `feat/session-present-counter`  
> Commit : `ad80b3caa7d5537b58857ba9e97cabc40ffa34be`  
> PR : [attendee-ems-back#40](https://github.com/Rabiegha/attendee-ems-back/pull/40)  
> Merge `staging` : `c75febddfa77255766df134db280f2af5f54b781` — CI/CD verts

## Problème supprimé

Le scan session calculait la présence avec deux agrégats sur tout l'historique : nombre de `IN`
admis moins nombre de `OUT` admis. Ces lectures avaient deux défauts :

- coût O(n) croissant avec `session_scans` ;
- course sur la dernière place, car deux QR différents n'étaient pas protégés par le même verrou.

Le hot path appelait aussi `findOne`, qui recalculait les visiteurs uniques. Un scan déclenchait donc
un troisième calcul historique sans rapport avec la décision immédiate.

## Invariant retenu

`sessions.present_count` représente le nombre de participants dont le **dernier scan admis** dans la
session est `IN`. Les scans rejetés ne comptent jamais. PostgreSQL reste la source de vérité :

1. la migration ajoute puis backfille le compteur avec l'ordre déterministe `scanned_at DESC, id DESC` ;
2. un index partiel couvre la recherche du dernier scan admis ;
3. l'insertion d'un `IN` réserve atomiquement une place sur la ligne session ;
4. un `OUT` décrémente sans passer sous zéro ;
5. suppression, annulation et cascade rétablissent l'état dérivé de l'historique ;
6. les champs qui définissent l'état d'un scan sont immuables : une correction passe par l'undo ;
7. une contrainte interdit tout compteur négatif.

Deux participants différents qui prennent la dernière place arrivent sur le même `UPDATE sessions
... WHERE present_count < checkin_capacity` : exactement un scan est admis. Le verrou advisory par
participant reste utile pour rendre deux scans simultanés du même QR idempotents.

## Contrat mobile/offline

Le backend conserve le code HTTP 201 compatible, mais ajoute une information non ambiguë :

- nouvelle admission : `duplicate: false` ;
- rejeu du même état : `duplicate: true` et aucun nouveau scan ;
- salle pleine : tentative enregistrée en `rejected_capacity`, réponse métier 400 `Session is full`.

Le mobile doit encore traduire `duplicate: true` en retour « déjà scanné » clair pendant le replay
offline. L9.1 fournit le contrat serveur ; il ne remplace pas la déduplication de la queue mobile.

## Undo et historique

Supprimer le dernier `IN` décrémente. Supprimer le dernier `OUT` restaure le `IN` précédent. Cette
dernière correction est refusée dans l'API si un autre participant a repris la place entre-temps.
Les cascades restent fidèles à l'historique, même si une jauge a été abaissée après les scans.

Le nombre de visiteurs uniques post-event reste distinct : toute personne ayant eu au moins un
`IN admitted` est un visiteur, même si elle est ressortie. L9.1 accélère la présence **actuelle**.

## Exploitation

Le contrôle est obligatoirement limité à une session, un event ou une organisation :

```text
npm run maintenance:session-presence -- --session-id <uuid>
npm run maintenance:session-presence -- --event-id <uuid>
npm run maintenance:session-presence -- --org-id <uuid>
```

La réparation exige explicitement :

```text
--apply --confirm RECONCILE_SESSION_PRESENCE
```

Le rapport indique aussi les sessions dont la présence dérivée dépasse la capacité effective, sans
les modifier silencieusement.

## Trois cycles de validation locale

- L9.1 ciblé : **9/9** ;
- régressions combinées H + L9.b + L9.a + L9.1 : **52/52** ;
- E2E complet, relancé deux fois : **87/87** ;
- unitaires : **82/82** ;
- build et typecheck : OK ;
- lint : 0 erreur, 55 avertissements préexistants ;
- base vierge : **90 migrations**, seed complet, `presence_mismatches=0` ;
- seconde répétition de migration après review : index final présent ;
- chaque base temporaire a été supprimée et vérifiée absente ;
- aucun secret ajouté ; aucune action production.

Cas couverts : IN, OUT, doublons, douze scanners du même QR, deux QR différents sur la dernière
place, capacité physique distincte, rejet sans choix, undo, place reprise, suppression en cascade,
scan historique et contrainte de non-négativité.

## Publication et mesure suivante

La PR #40 a été mergée et déployée sur staging avec migration appliquée et health exact. Les tests
prouvent la correction du compteur, pas encore le débit check-in LFD.

La campagne inscription L9.b/L9.a est désormais terminée : 70/s soutenues et choc 250 validés avec
invariants, voir le
[rapport final](../session-travail-autonome/rapports-load-tests/2026-07-20-1635-1708-l9b-atomique-campagne-finale.md).
Il reste à mesurer séparément le **check-in L9.1**, puis le scénario combiné, dans une nouvelle
fenêtre de charge autorisée. Le choc 3 000 simultanés reste distinct d'un débit soutenu et exigera
plusieurs générateurs synchronisés pour constituer une preuve crédible.
