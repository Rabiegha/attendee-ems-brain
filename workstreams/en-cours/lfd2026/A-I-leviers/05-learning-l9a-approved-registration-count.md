# 05 — Learning L9a : capacité event atomique et transaction courte

> Date : 2026-07-20  
> Branche back : `feat/register-event-tx-slim`  
> Commit : `873a08f`  
> PR : [attendee-ems-back#39](https://github.com/Rabiegha/attendee-ems-back/pull/39)  
> Correctif verrou différé : [attendee-ems-back#42](https://github.com/Rabiegha/attendee-ems-back/pull/42)  
> Merge initial `staging` : `4253d3993670f8714c43c070dcbeaef967370166`

## But

L9a traite le formulaire event classique :

```text
POST /api/public/events/:publicToken/register
```

Avant L9a, la transaction exécutait un `COUNT(*)` des inscriptions approuvées avant de créer ou
restaurer l'inscription. Le coût grandissait avec la table et ce contrôle ne constituait pas une garde
atomique commune aux écritures publiques, admin, import et changements de statut.

## Invariant retenu

La table `events` porte désormais `approved_registration_count` : nombre exact des registrations :

- `status = approved` ;
- `deleted_at IS NULL`.

Une migration :

1. ajoute et backfille le compteur ;
2. interdit une valeur négative ;
3. maintient le compteur sur `INSERT`, `UPDATE` et `DELETE` de `registrations` ;
4. réserve une place avec un `UPDATE events ... WHERE count < capacity` atomique ;
5. annule toute l'écriture avec `event_capacity_exceeded:<eventId>` si la jauge est pleine.

La transaction PostgreSQL annule aussi le mouvement de compteur si une écriture ultérieure échoue.
Une restauration, un refus, une annulation, un soft-delete ou un changement d'event ne peuvent donc
pas laisser une demi-réservation.

## Transaction publique raccourcie

`registerToEvent` charge maintenant le contexte d'affichage et d'email avant de réserver une
connexion transactionnelle. Dans la transaction restent uniquement :

- relecture de l'état mutable de l'event et de l'auto-approbation ;
- upsert attendee ;
- anti-doublon ;
- create/restore registration ;
- invariants de choix de sessions.

Le `registration.count` a disparu. Les emails, notifications admin et recalculs attendee ne sont
planifiés qu'après le commit. Les erreurs de jauge et d'unicité reviennent en `409` métier propre.

Les chemins admin `create`, `approve`, `restore`, import et bulk utilisent aussi le compteur O(1).
Le trigger reste la dernière garde concurrente, quelle que soit l'origine de l'écriture.

## Exploitation

Commande de contrôle, obligatoirement limitée à un event ou une organisation :

```text
npm run maintenance:event-counts -- --event-id <uuid>
npm run maintenance:event-counts -- --org-id <uuid>
```

Le mode écriture exige en plus :

```text
--apply --confirm RECONCILE_EVENT_COUNTS
```

## Régression et concurrence

La suite L9a couvre :

- nominal et snapshots attendee ;
- capacité atomique sous concurrence ;
- retries concurrents du même email ;
- endpoint session LFD lorsque la capacité event est pleine ;
- statuts `awaiting`, `approved`, `refused` ;
- approbations admin concurrentes ;
- soft-delete et restauration ;
- event sans capacité ;
- event non publié sans écriture partielle ;
- rollback avant tout effet email.

Preuves locales avant publication :

- unitaires : **82/82** ;
- E2E L9a : **12/12** ;
- E2E complet : **78/78** ;
- lint CI : 0 erreur ;
- typecheck, build et `prisma validate` : OK ;
- migration + seed E2E rejoués depuis une base PostgreSQL vierge : OK ;
- audit de cette base : **0 compteur divergent, 0 event au-dessus de sa capacité** ;
- scan du diff : aucun secret détecté.

## Incident évité pendant la review

Le premier rejeu sur base vierge a montré que les seeders de démonstration généraient aléatoirement
plus d'`approved` que la capacité d'un event. Le nouveau trigger a correctement refusé ces données,
ce qui aurait rendu la CI rouge. Les seeders basculent désormais l'excédent en `awaiting`, puis le
rejeu complet sur base vierge est passé.

Autre faux négatif éliminé : Supertest ouvrait un listener éphémère pour des requêtes concurrentes,
ce qui provoquait ponctuellement `ECONNRESET`. Les suites de compteurs gardent maintenant un listener
réel pendant leur exécution ; la suite complète est redevenue stable sans diminuer la concurrence.

## Lecture performance

L9a supprime un `COUNT(*)` et réduit la durée de possession d'une connexion. Le compteur event reste
une ligne partagée et sérialise brièvement la réservation exacte d'une place : c'est le coût nécessaire
pour garantir la jauge sur une instance comme sur plusieurs writers. La décision de capacité LFD ne
sera pas déduite des tests E2E ; elle vient des mesures k6 après L9.1, sur le SHA réellement déployé.

La première campagne a ensuite montré que réserver la capacité event avant certaines lectures
préparatoires conservait le verrou event trop longtemps pendant le flux session. La PR #42 diffère
la réservation au dernier moment utile. Après la correction L9.b atomique #50, le compteur event est
resté exact pendant **8 400 inscriptions à 70/s** ; cette mesure intégrée ne remplace pas un scénario
k6 dédié au formulaire event si LFD utilise aussi cet endpoint.

## Publication et déploiement staging

- CI de la PR #39 : [run 29730719426](https://github.com/Rabiegha/attendee-ems-back/actions/runs/29730719426), quatre jobs verts ;
- CI du push `staging` : [run 29731563698](https://github.com/Rabiegha/attendee-ems-back/actions/runs/29731563698), quatre jobs verts ;
- CD staging : [run 29731944618](https://github.com/Rabiegha/attendee-ems-back/actions/runs/29731944618), vert ;
- SHA réellement déployé et servi : `4253d3993670f8714c43c070dcbeaef967370166` ;
- image : `sha256:21f8953c08d2baa95d2bc87becd5d7bb5c6c1841d4db98559969e33b98833280` ;
- migration appliquée : `20260720040000_event_approved_registration_count` ;
- health externe : API, PostgreSQL, Redis et migrations `ok`, version exacte confirmée ;
- garde prod strictement en lecture seule : health prod resté vert, aucune mutation ni aucun déploiement prod.
