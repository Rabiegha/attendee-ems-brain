# 02 — Cadrage reprise chantier A — L9a / L9b / L9.1

> Date : 2026-07-17  
> Objet : reprendre le chantier A proprement sur les leviers L9a, L9b et L9.1, depuis `staging` cote back, sans repartir de l'archive ni ramener un fourre-tout.

## 1. Etat actuel

Le chantier A est en cours. La partie fondation a deja ete posee et mesuree :

- L1/L2/L10 : stack staging, pgBouncer et pools — deja merges sur `staging` et mesures le 2026-07-10.
- L9a : transaction d'inscription event allegee (`registerToEvent`) — non code sur la branche propre actuelle.
- L9b : transaction d'inscription session allegee (`registerToSession`) — non code, **prioritaire LFD** car utilisee par `back-office-event`.
- L9.1 : compteur de presence session O(1) — cadre, priorise, non code.
- L3 : `directUrl` Prisma — a traiter **avant L9b/L9a/L9.1** pour eviter les migrations Prisma via pgBouncer.
- L7/L8 : restent a traiter apres L9b/L9a/L9.1.

La branche back de travail creee pour cette reprise est pour l'instant :

- `feat/register-tx-slim-and-present-counter`, creee depuis `staging` au commit `da5b118`.

Si on garde la separation stricte jusqu'aux PR, elle sera decoupee ou remplacee par :

- `feat/register-session-tx-slim` pour L9b ;
- `feat/register-event-tx-slim` pour L9a ;
- `feat/session-present-counter` pour L9.1.

Cote brain, les notes restent sur `main`. Les branches dediees depuis `staging` concernent le code back.

## 2. Pourquoi L9 est prioritaire

Le plafond mesure autour de 33 inscriptions/s ne semble pas venir du CPU, du pool DB ou de l'email. Le goulot principal est dans le chemin d'inscription public :

- fichier cible : `src/modules/public/public.service.ts`
- methodes cibles : `registerToEvent` **et** `registerToSession`
- symptome commun : la transaction tient la connexion pendant des controles capacite par `COUNT`/verrou, puis l'upsert attendee et la creation/reutilisation de registration.

L9 doit raccourcir cette transaction et deplacer le controle capacite vers une operation atomique plus directe. L'objectif est d'eviter que les inscriptions concurrentes se serialisent sur une transaction trop large.

### L9a — endpoint event classique

Endpoint :

```txt
POST /api/public/events/:publicToken/register
```

Methode :

```txt
PublicService.registerToEvent(publicToken, dto)
```

Probleme :

- controle capacite event par `registration.count` dans la transaction ;
- upsert attendee + anti-doublon + create/restore registration dans le meme chemin ;
- risque de serialisation sous pic sur les events a capacite.

Ce levier reste important pour les formulaires publics event classiques. Il ne doit pas disparaitre.

### L9b — endpoint session LFD reel

Endpoint réellement utilise par `back-office-event` :

```txt
POST /api/public/events/sessions/:sessionToken/register
```

Methode :

```txt
PublicService.registerToSession(sessionToken, dto)
```

Probleme :

- verrou pessimiste `FOR UPDATE` sur la ligne `sessions` ;
- controle capacite session par `registrationSessionChoice.count` ;
- recalcul stats live par `registrationSessionChoice.count` ;
- la methode cree/reutilise bien une registration event, mais le goulot LFD est surtout la capacite session.

Pour LFD, L9b est le premier hot path a traiter parce que le formulaire `back-office-event` passe par ce endpoint.

Regle de reprise :

- partir du `staging` propre actuel ;
- regarder l'archive uniquement comme reference de lecture si necessaire ;
- rejouer a la main le diff minimal ;
- ne pas cherry-pick une branche archive/fourre-tout ;
- ajouter un test de non-regression avant mesure.

## 3. Ce que L9 doit garantir

L9 touche du code metier sensible. Les invariants a conserver :

- aucune double inscription pour le meme attendee/event ;
- aucune double inscription pour le meme couple registration/session ;
- pas de depassement de capacite event ;
- pas de depassement de capacite session ;
- statut `confirmed` / `waitlisted` correct sur `RegistrationSessionChoice` ;
- meme comportement pour les statuts `approved`, `awaiting`, `refused` ;
- les snapshots attendee/registration restent identiques ;
- les emails/PDF et effets de bord restent asynchrones et declenches apres l'inscription, jamais dans la transaction critique ;
- les erreurs fonctionnelles restent propres : doublon, event complet, event non publie, etc.

Mesure attendue apres L9 :

- test local de non-regression ;
- deploiement staging depuis la branche/MR ;
- k6 sur staging, pas en local ;
- comparaison avec les mesures precedentes sur les deux endpoints si possible ;
- test de charge dedie L9b sur `/public/events/sessions/:sessionToken/register` ;
- si possible, test de rupture plus agressif que le run doux a ~20 req/s.

## 4. Pourquoi L9.1 arrive juste apres

L9.1 vise un autre goulot : le check-in session. Le probleme identifie est dans :

- fichier cible : `src/modules/sessions/sessions.service.ts`
- methode de scan session ;
- symptome performance : a chaque scan IN admis, le code recompte `session_scans` avec un `COUNT` IN et un `COUNT` OUT ;
- symptome correction confirme par l'audit du 19/07 : les `COUNT` sont avant la transaction et
  l'advisory lock est derive de `sessionId:registrationId`, donc il ne serialise pas deux participants
  differents qui prennent simultanement la derniere place.

Ce calcul est O(n) : plus la table grossit, plus le cout grandit. L9.1 remplace ce recalcul par un compteur O(1).

Option par defaut :

- ajouter une colonne Postgres de type `present_count` sur `sessions` ;
- backfiller selon le **dernier scan admis par participant/session**, pas avec `IN - OUT` brut ;
- incrementer sur un IN admis avec une garde atomique `present_count < checkin_capacity` ;
- decrementer sur un OUT admis sans passer sous zero ;
- garder l'operation dans la meme transaction que le scan pour rester coherent.

Redis n'est pas obligatoire pour L9.1. Il peut devenir interessant plus tard pour l'affichage live ou un compteur derive, mais la verite simple et durable pour ce levier reste Postgres.

## 5. Ce que L9.1 doit garantir

Invariants a conserver :

- un scan IN admis augmente la presence ;
- un scan OUT admis la diminue ;
- un scan rejete ne change pas la presence ;
- un double scan/idempotence ne produit pas un compteur faux ;
- le compteur ne passe pas sous zero ;
- la capacite de check-in distincte (`checkin_capacity`) reste respectee ;
- la capacite d'inscription session et la capacite physique restent deux notions distinctes.
- deux QR differents concurrents sur la derniere place produisent exactement une admission ;
- deux scanners du meme QR produisent une admission et un resultat doublon explicite, exploitable
  pendant le replay offline mobile.
- le dashboard J-ENTREES lit la présence actuelle O(1), tandis que le bilan post-event considère
  présent toute personne ayant eu au moins un `IN admitted`, même si elle est ensuite sortie.

Mesures attendues apres L9.1 :

- tests unitaires ou integration sur IN/OUT/rejet/double scan et H-T20→H-T25 ;
- verification migration + backfill initial du compteur ;
- test k6 combine si possible : inscriptions + check-ins session + check-ins entree.

## 6. Organisation proposee

Ordre de travail recommande :

1. L3 `directUrl` Prisma : schema + env docs, test `prisma validate`, commit propre.
2. L9b session : diff minimal sur `registerToSession`, test local, commit propre.
3. Mesure L9b sur staging avec le endpoint reel `back-office-event`.
4. L9a event : diff minimal sur `registerToEvent`, test local, commit propre.
5. Mesure L9a sur staging avec le script event classique.
6. L9.1 seul : migration + code + tests, commit propre.
7. Mesure L9.1, idealement sur scenario combine.
8. Mise a jour du suivi chantier A avec resultats et decisions.

La branche actuelle regroupe L9a/L9b et L9.1 pour preparer la reprise, mais on garde des commits distincts. Si le diff grossit trop, on separe en branches :

- `feat/register-session-tx-slim` (L9b)
- `feat/register-event-tx-slim` (L9a)
- `feat/session-present-counter`
- `chore/prisma-directurl` (L3)

## 7. Points de vigilance avant de coder

- `staging` contient deja des changements recents de sessions publiques, waitlist, stats live et capacite check-in distincte. Ne pas les ecraser.
- `back-office-event` utilise le endpoint session, pas le endpoint event classique : ne pas mesurer uniquement `registerToEvent`.
- Le throttling QR/Redis peut fausser les prochains tests k6 : verifier les limites staging avant mesure.
- Prisma migrations via pgBouncer ont deja declenche un P1002 advisory lock. Tant que L3 `directUrl` n'est pas pose, lancer les migrations avec prudence.

## 8. Definition de fini pour cette reprise

La reprise L9b/L9a/L9.1 est consideree propre si :

- L9a, L9b et L9.1 sont codes avec diffs revisables ;
- chaque levier a son test de non-regression ;
- chaque levier a son commit propre ;
- staging est mesure apres chaque changement significatif ;
- le tableau de suivi est mis a jour avec commit, MR, resultat k6 et decision suivante.
