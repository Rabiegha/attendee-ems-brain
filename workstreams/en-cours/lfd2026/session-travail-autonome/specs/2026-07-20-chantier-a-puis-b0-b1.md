# Spec de session autonome — Chantier A puis B0/B1

> **Date de cadrage :** 2026-07-20
> **Horizon :** session prolongée, jusqu'à atteinte de la Definition of Done ou blocage réel
> **Environnements autorisés :** local, CI et staging
> **Environnements interdits sans nouvel accord :** `main` et production
> **Workstreams :** A/I/L9, puis B0/B1

## 1. Objectif

Conduire les leviers restants du chantier A de manière séquentielle, mesurable et réversible, puis
terminer B0/B1. Chaque lot doit être codé, relu plusieurs fois, corrigé, testé, publié dans une PR
isolée, validé par la CI, déployé sur staging et mesuré lorsque le lot touche la performance.

La session ne doit pas produire une accumulation de code « probablement correct ». Elle doit produire
une suite de changements petits, prouvés, documentés et comparables.

## 2. Résultat attendu

À la fin de la session :

- L9.b possède une mesure k6 reproductible sur le vrai endpoint session LFD ;
- L9.a est livré avec ses régressions event/capacité/concurrence ;
- L9.1 garantit présence et capacité session atomiques sous plusieurs scanners ;
- L7/L8 sont audités contre C2/C2.1, puis clos comme déjà couverts ou implémentés sans doublon ;
- B0 sait appeler Gotenberg Cloud Run privé depuis le backend staging ;
- B1 rend le chemin PDF exploitable sous timeout, panne et concurrence ;
- chaque branche a un commit et une PR propres, une CI verte et une preuve staging ;
- les mesures, anomalies, reviews et décisions sont consignées dans les fichiers de suivi ;
- aucun changement de cette session n'est déployé en production.

## 3. Ordre d'exécution

L'ordre par défaut est strict. Une étape ne commence pas tant que la précédente n'a pas un résultat
explicable : livrée, abandonnée avec preuve ou bloquée par un prérequis identifié.

1. **Préflight A** : état `staging`, migrations, CI/CD, scripts k6, fixtures et accès.
2. **L9.b — mesure** : débloquer le scénario session, établir le résultat actuel et documenter.
3. **L9.a — code + régressions + mesure**.
4. **L9.1 — code + migration/backfill + concurrence multi-scanner + mesure**.
5. **L7 — audit C2** : ne coder que le delta réellement manquant autour de BullMQ.
6. **L8 — audit/process worker** : séparer le worker si le runtime actuel ne le garantit pas déjà.
7. **Revue de clôture A** : cohérence des leviers, résultats k6 et suivi.
8. **B0 — client Gotenberg OIDC staging**.
9. **B1 — durcissement PDF**.
10. **Revue de clôture B0/B1** : CI/CD staging, tests d'échec et documentation.

Si le credential Gotenberg n'est pas encore disponible après A, préparer B0/B1 sans faux secret,
documenter précisément le blocage et poursuivre uniquement les travaux qui restent vérifiables.

## 4. Stratégie de branches

Chaque branche part du dernier `origin/staging` validé après merge de la branche précédente.

| Lot | Branche prévue | Contenu autorisé |
| --- | --- | --- |
| L9.b mesure | `test/k6-register-session-l9b` si le harnais doit changer | fixtures et scripts k6 uniquement |
| L9.a | `feat/register-event-tx-slim` | transaction event, compteur/garde et tests associés |
| L9.1 | `feat/session-present-counter` | migration, backfill, scan atomique et tests |
| L7 | `feat/email-async-bullmq` seulement si gap confirmé | delta email async non déjà couvert par C2 |
| L8 | `feat/process-role-worker` seulement si gap confirmé | gating API/worker et démarrage séparé |
| B0 | `feat/gotenberg-oidc-client` | appel privé Cloud Run, config et smoke staging |
| B1 | `feat/pdf-generation-hardening` | timeout, retry, fallback, observabilité et charge |

Règles Git :

- aucun développement directement sur `staging` ;
- commits courts et intentionnels ;
- aucun reformatage massif mélangé à la logique ;
- aucune branche fourre-tout A+B ;
- l'archive `staging-archive-2026-06-25` est consultable en lecture ;
- aucun cherry-pick global depuis l'archive ; le diff minimal est rejoué manuellement ;
- la branche n'est mergée sur `staging` qu'après reviews, tests locaux et CI verte.

## 5. Boucle obligatoire code → review → correction

Chaque lot passe au minimum par quatre passes. Une correction relance les tests concernés puis la
review qui l'a déclenchée.

### Review 1 — métier et régressions

- invariants avant/après explicités ;
- statuts, idempotence, capacité, waitlist et erreurs inchangés sauf décision écrite ;
- aucun effet secondaire critique déplacé avant le commit métier ;
- tests capables d'échouer sur l'ancien bug ou la course visée.

### Review 2 — concurrence et performance

- transaction réduite au strict nécessaire ;
- verrou partagé par la bonne ressource métier ;
- absence de `COUNT` ou requête O(n) sur le hot path lorsque le levier vise à les enlever ;
- comportement vérifié sous plusieurs requêtes simultanées ;
- comparaison k6 faite avec les mêmes données et paramètres.

### Review 3 — sécurité, isolation et migrations

- isolation organisation/event ;
- permissions et erreurs sans fuite de données ;
- migration additive, backfill correct et stratégie de rollback ;
- secrets absents du code, des logs, des fixtures et des commits ;
- timeout et erreurs externes bornés.

### Review 4 — diff final et exploitabilité

- diff relu fichier par fichier ;
- pas de code mort, duplication, TODO caché ou changement hors scope ;
- CI, CD staging, healthchecks et observabilité vérifiés ;
- documentation fidèle au SHA réellement déployé ;
- risques résiduels et décision suivante écrits avant merge du lot suivant.

## 6. Régressions obligatoires L9

### L9.b — inscription session

- première inscription `confirmed` et incrément unique de `registered_count` ;
- nouvelle tentative idempotente sans double choix ni double incrément ;
- capacité atteinte : `waitlisted` ou refus conforme à la configuration ;
- plusieurs inscriptions concurrentes : aucun dépassement de capacité ;
- annulation, suppression, restauration, import/admin et changement de statut réconcilient le compteur ;
- aucune régression du maximum de deux activités `confirmed` par jour ;
- les effets email/billet restent après le commit et hors transaction critique.

### L9.a — inscription event

- inscription nominale et snapshot attendee inchangés ;
- même email/personne : aucune double registration ;
- capacité event jamais dépassée sous concurrence ;
- comportements `approved`, `awaiting`, `refused`, restauration et événement non publié conservés ;
- erreur métier propre, sans réservation partielle ;
- événement sans capacité inchangé fonctionnellement ;
- aucun email/PDF exécuté dans la transaction de réservation.

### L9.1 — présence et scan session

- premier `IN admitted` : `present_count + 1` ;
- `OUT admitted` : `present_count - 1`, jamais sous zéro ;
- rejet, doublon et replay : compteur inchangé ;
- N requêtes du même QR : une seule admission et un contrat doublon explicite ;
- N QR différents sur la dernière place : une seule admission supplémentaire ;
- `checkin_capacity` distincte de `capacity` respectée ;
- undo d'un scan réconcilie correctement l'état ;
- backfill fondé sur le dernier scan admis par participant/session ;
- export post-event : une personne entrée puis sortie reste participante ;
- le dashboard J-ENTREES converge vers la vérité serveur après reconnexion.

## 7. Tests et gates par branche

Avant PR :

1. tests unitaires ciblés ;
2. tests d'intégration/E2E métier ;
3. tests de concurrence lorsque le lot touche une capacité ou un compteur ;
4. validation Prisma/migration pour L9.1 ;
5. lint, typecheck et build du périmètre ;
6. `git diff --check` et inspection du diff complet.

Après push :

1. PR créée avec description, invariants, tests et rollback ;
2. CI entièrement verte ;
3. corrections des findings de review ;
4. nouvelle CI verte après le dernier commit ;
5. merge sur `staging` ;
6. CD staging terminé ;
7. healthcheck et version/SHA staging vérifiés ;
8. smoke fonctionnel avant toute charge.

Une CI rouge n'est jamais contournée. Une erreur présentée comme « flaky » doit être reproduite ou
justifiée par une preuve avant relance/merge.

## 8. Protocole k6

### Principe de comparaison

Chaque comparaison avant/après conserve :

- même endpoint et même payload ;
- même type et nombre de VUs ;
- mêmes durée, ramp-up et profil de burst ;
- même événement/session, capacité et statut publié ;
- même chemin réseau et même environnement staging ;
- même méthode de nettoyage/préparation des données ;
- dashboard et monitoring ouverts pour corréler API, PostgreSQL, Redis, CPU et mémoire.

### Mesures consignées

- nombre de tentatives et résultats métier ;
- succès techniques, `4xx` attendus et `5xx` ;
- médiane, moyenne, p95 et p99 ;
- débit par seconde et durée d'absorption ;
- connexions/pool DB, CPU, RAM, saturation et timeouts ;
- surbooking, doublons ou dérive des compteurs ;
- SHA backend, version du script, date/heure et configuration du scénario.

### Gates non négociables

- zéro surbooking ;
- zéro corruption/dérive de compteur ;
- zéro `5xx` inexpliqué ;
- résultats métier cohérents avec la capacité/waitlist ;
- comparaison chiffrée avant/après avant de déclarer un levier efficace ;
- aucun test de charge dirigé vers la production.

Les seuils contractuels de latence p95/p99 et l'interprétation exacte des « 3 000 simultanées » sont
à faire valider. Sans réponse, la session mesure et documente, mais ne fabrique pas un GO contractuel.

## 9. B0 — Definition of Done

- appel Gotenberg privé via `google-auth-library.getIdTokenClient(...)` ;
- audience et URL configurées par environnement, sans valeur secrète commitée ;
- l'appel anonyme reste refusé ;
- smoke staging produisant un PDF valide (`%PDF`, non vide, contenu attendu) ;
- timeout explicite et erreur structurée ;
- fallback Puppeteer conservé si c'est la décision du chantier ;
- aucune génération PDF dans la transaction d'inscription ;
- logs sans credential ni contenu personnel inutile ;
- tests du chemin succès, 401/403, timeout et indisponibilité.

## 10. B1 — Definition of Done

- timeout/retry bornés et réservés aux erreurs transitoires ;
- idempotence : un job billet ne génère/envoie pas plusieurs billets par erreur ;
- fallback testé et observable ;
- nettoyage des fichiers temporaires/mémoire ;
- métriques succès, durée, fallback, retry et échec final ;
- concurrence représentative sans saturation incontrôlée de l'API ;
- reprise d'un job échoué et état final utilisateur cohérent ;
- smoke C2.1 : réservation rapide, job PDF, puis job email ;
- CI/CD staging et rapport de test joints à la PR.

## 11. Documentation obligatoire après chaque lot

Mettre à jour au minimum :

- `A-I-leviers/01-suivi-leviers.md` ;
- le learning propre au levier ou un nouveau learning daté ;
- `03-suivi-chantiers.md` ;
- `00-plan-action.md` uniquement si périmètre, ordre ou estimation changent ;
- les specs/tests H/J lorsque L9.1 modifie leur contrat ;
- le suivi B0/B1 et C2.1 lorsque le PDF est concerné.

Pour chaque lot, noter : branche, commits, PR, fichiers majeurs, invariants, tests locaux, résultat CI,
résultat CD, SHA staging, scénario k6, métriques avant/après, findings des reviews, corrections et
risques restants.

Voir également
[`2026-07-20-b0-b1-gotenberg-entrees.md`](2026-07-20-b0-b1-gotenberg-entrees.md) pour la configuration
non secrète de Gotenberg et la procédure de transfert du credential hors Git.

## 12. Autorisations nécessaires pour lancer la session prolongée

### Confirmation de périmètre

Une confirmation explicite `GO session autonome A → B0/B1` doit autoriser :

- création et push de branches de travail dédiées ;
- création des PR ;
- correction itérative du code et des tests dans le périmètre de la spec ;
- merge des PR vers `staging` après CI verte ;
- déclenchement et suivi du CD staging ;
- création et nettoyage de données **strictement loadtest** sur staging ;
- exécution de k6 sur staging aux créneaux validés ;
- mise à jour et publication des documents de suivi.

Cette autorisation n'inclut jamais `main`, la production, la suppression de données métier réelles,
la rotation de secrets, un changement d'infrastructure majeur ou le contact d'un tiers.

### Règle de fin de session et reprise inter-poste

Avant de terminer chaque période de travail :

- commiter et pousser tout code, test, migration, documentation et rapport non sensible ;
- ne laisser aucune modification utile uniquement dans le worktree local ;
- publier un point de reprise indiquant branche, dernier commit, PR, état CI/CD, étape atteinte,
  prochaine action et blocages ;
- vérifier que les secrets, tokens, exports et fixtures contenant des données personnelles ne sont ni
  suivis par Git ni présents dans le diff ou l'historique ;
- transférer séparément les secrets encore requis par un canal sécurisé ;
- ne jamais contourner cette séparation au motif que la session reprend sur un autre poste.

Le critère « tout doit être commité » signifie donc **tout ce qui est partageable et non sensible**.
Un secret requis doit être provisionné hors Git sur le poste suivant ou directement dans staging.

### Accès/valeurs bloquants

Pour L9.b/L9.a/L9.1 :

1. un compte staging valide avec les droits nécessaires à la préparation des fixtures ;
2. un événement de load test publié et isolé ;
3. au moins un `SESSION_TOKEN` dédié pour L9.b ;
4. la possibilité de réinitialiser les inscriptions/scans de cet événement de test ;
5. la fenêtre pendant laquelle une charge importante sur staging est autorisée ;
6. confirmation des seuils attendus, ou acceptation du mode « mesure sans GO contractuel ».

Pour B0/B1 :

1. le credential/service account autorisé à invoquer Gotenberg Cloud Run, posé par canal sécurisé dans
   les secrets staging — jamais dans ce dépôt ou dans une conversation ;
2. l'URL Cloud Run et l'audience OIDC exactes confirmées dans la configuration staging ;
3. le droit de déclencher un smoke PDF et un test de concurrence raisonnable sur le service ;
4. la décision sur `min-instances=1` pendant les pics/event, ou l'autorité pour seulement mesurer le
   cold start sans modifier cette configuration.

GitHub et le push sont actuellement disponibles dans l'environnement. La session est autorisée à
créer ses propres comptes, événements et tokens de load test sur staging. Le credential Gotenberg a
été reçu et vérifié localement ; son injection sécurisée sur staging reste à confirmer lors de B0.

### Autorisations staging reçues le 2026-07-20

Le responsable du projet autorise, exclusivement sur **staging** :

- la création des comptes techniques/de test nécessaires aux scénarios L9 ;
- la création d'un événement LFD de charge isolé et des sessions associées ;
- la génération des tokens de session dédiés aux tests ;
- la création des inscriptions, scans et autres fixtures synthétiques nécessaires ;
- le nettoyage des seules données créées pour ces tests ;
- l'exécution des tests de charge k6 pendant la fenêtre nocturne autorisée.

Garde-fous impératifs :

- vérifier l'URL, l'environnement et l'identité de la base avant toute création, charge ou suppression ;
- marquer clairement chaque fixture comme donnée de load test ;
- conserver la liste ou le préfixe des identifiants créés afin de cibler le nettoyage ;
- refuser automatiquement toute URL ou configuration pointant vers la production ;
- ne supprimer aucune donnée staging antérieure ou non créée par cette session ;
- ne jamais utiliser un compte, un événement ou un token de production ;
- demander une nouvelle fenêtre si la charge est reportée hors de la période nocturne autorisée.

La production, `main` et toute donnée réelle restent explicitement hors périmètre. Cette autorisation
ne vaut pas encore validation contractuelle des performances : elle autorise à préparer, exécuter,
mesurer et optimiser les tests sur staging.

## 13. Conditions d'arrêt ou d'escalade

La session continue de façon autonome tant qu'un travail sûr et vérifiable reste possible. Elle
s'arrête avant l'action concernée et demande une décision si :

- un secret ou accès requis manque et aucune alternative locale/staging sûre n'existe ;
- une migration risque de perdre ou réécrire des données non dédiées au load test ;
- les invariants métier sont ambigus et les deux interprétations changent le résultat utilisateur ;
- la seule solution restante exige production, `main`, multi-instance, Redis obligatoire ou nouvelle
  infrastructure non autorisée ;
- la CI révèle une régression hors périmètre impossible à corriger sans élargissement significatif ;
- k6 risque de toucher un service partagé non isolé ou de perturber des utilisateurs réels ;
- B0 nécessite une action GCP ou une rotation de secret qui n'a pas été autorisée.

Un échec de test n'est pas à lui seul un arrêt : il déclenche diagnostic, correction, review et
nouvelle exécution.

## 14. Points de contrôle communiqués

Un compte rendu est produit :

- après le préflight ;
- après chaque boucle review/correction importante ;
- à l'ouverture de chaque PR ;
- après CI et CD staging ;
- après chaque mesure k6 ;
- lors d'un blocage nécessitant une autorisation ;
- à la clôture de A, puis à la clôture de B0/B1.

Le compte rendu final distingue toujours : fait, testé localement, validé CI, déployé staging,
mesuré, bloqué et reporté. Aucun de ces états ne doit être résumé abusivement par « terminé ».
