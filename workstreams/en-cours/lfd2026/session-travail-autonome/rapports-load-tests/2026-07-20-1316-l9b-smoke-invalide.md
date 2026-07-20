# Load test — L9.b — smoke initial invalidé

## Verdict rapide

- Statut : **INVALIDE pour la chronologie et les ressources**
- Niveau LFD : non applicable
- Conclusion : l'inscription unique est correcte, mais la mise en veille du poste a créé une pause de
  35 minutes et le script de surveillance contenait des expressions `awk` non portables sur macOS.

## Identité et objectif

- Date Europe/Paris : 2026-07-20, 13:16–13:52
- Environnement : staging uniquement ; production contrôlée en lecture seule
- Endpoint : `POST /api/public/events/sessions/<token>/register`
- Backend : `c75febddfa77255766df134db280f2af5f54b781`
- Harnais : `2e4b5f495c9aadbb7a9d955dd160de02c3da4526`
- k6 : 2.1.0 ; scénario `register-session-smoke.js`
- But : valider la fixture et les invariants avant toute charge.

## Profil et résultats

- 1 VU, 1 identité et 1 requête ; fixture `[LOADTEST LFD]`, capacité 500 000, waitlist inactive.
- Résultat : **1/1 confirmed**, 0 erreur, 0 itération perdue.
- Latence : 181,78 ms (médiane/moyenne/p95/p99/max identiques).
- DB : 1 registration, 1 email distinct, 1 choix session, compteur stocké = vérité réelle = 1.
- Surbooking/doublon/dérive : aucun.

## Invalidations et correction

- Le poste a dormi entre les phases du runner : les timestamps de récupération et ressources ne
  représentent pas un run continu.
- Le contrôle `awk` BSD a échoué après l'appel k6 ; l'appel et les invariants SQL restent factuels,
  mais le run complet n'est pas recevable comme preuve opérationnelle.
- Corrections : expressions `awk` rendues POSIX, commandes enveloppées dans `caffeinate -dimsu`, puis
  smoke rejoué sous un nouvel identifiant.

## Avis LFD et traçabilité

- Ce run prouve seulement que la fixture sait créer une inscription correcte.
- Il ne contribue à aucun GO performance.
- Artefacts locaux ignorés par Git :
  `attendee-ems-back/tmp/k6/lfd-l9-20260720-a/gen-01/`.
- Données synthétiques conservées temporairement dans la fixture pour les paliers suivants ; nettoyage
  global en fin de campagne.
