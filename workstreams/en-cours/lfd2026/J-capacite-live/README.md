# Chantier J — Capacité live forte charge

> Portier Redis (anti-survente) + WebSocket statut live + pic combiné inscription/check-in.
> Cadré dans le [workstream 02 — capacité live forte charge](../../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md).
> **Statut réel (audit code 2026-07-19) :** le flux public est déjà branché côté attendee sur une
> source de vérité en base + stats d'**inscription** live WebSocket (PR #27). Le tableau de bord des
> **entrées physiques** demandé par le CDC n'est que partiellement couvert ; il devient le lot
> `J-ENTREES`. Le portier Redis reste conditionné par la mesure.

## Décision d'architecture alignée avec C2.1

- La réservation et le check-in restent synchrones, courts et atomiques dans PostgreSQL.
- BullMQ ne met pas la décision métier en attente et n'accepte jamais une place après coup.
- Après commit, BullMQ traite seulement PDF, email, statistiques non critiques,
  notifications, WebSocket et autres effets secondaires.
- Le portier Redis est un levier conditionnel de contrôle/capacité, à activer uniquement si k6 le justifie ;
  il ne transforme pas la réservation en job asynchrone.
- Le multi-instance API reste reporté tant que le chantier F stateless n'est pas terminé, sauf nécessité
  démontrée par la mesure et arbitrage explicite.

Cette règle est la même que dans [C2.1](../C-migration-esp/C2-1-email-billet-session/00-suivi.md)
et dans le chantier coordinateur [N](../N-architecture-event-ready/README.md).

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (ligne J du tableau)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **J**)
- **Statut :** 🟡 en cours — recoupe H/I, ne pas double-compter (S30–S31) · implémentation livrée côté attendee (PR #27)

## J-ENTREES — tableau de bord des entrées LFD

### Ce que mesure chaque compteur

Le tableau de bord ne doit jamais utiliser `registered_count` comme preuve d'entrée :

| Mesure | Définition | Usage |
| --- | --- | --- |
| `registered_count` | choix de session au statut `confirmed` | réservations, places d'inscription restantes |
| `present_count` | personnes dont le **dernier scan admis** est `IN` | personnes actuellement dans la salle et taux de remplissage live |
| `unique_entered_count` | personnes ayant eu au moins un scan `IN` admis | total des participants réellement entrés au moins une fois |
| `fill_rate` | `present_count / checkin_capacity_effective × 100` | remplissage physique actuel ; `null` si aucune capacité n'est définie |

Un participant qui entre puis sort vaut donc `present_count = 0` mais reste compté dans
`unique_entered_count`. Dans l'export post-event, il est **présent**, car il a participé.

### Audit du code actuel

| Exigence CDC | État constaté | Verdict |
| --- | --- | --- |
| Nombre d'entrées | `SessionsService` calcule déjà la présence courante et les visiteurs uniques à partir des scans admis | 🟡 disponible côté API, mais coûteux et mal nommé `_count.scans` dans la liste |
| Remplissage par salle/créneau | le front affiche `_count.scans / session.capacity` | 🟠 proche, mais le bon dénominateur physique est `checkin_capacity ?? capacity` |
| Temps réel | `public-session:stats` ne diffuse que `registered/seats_left/state`; `scanParticipant` n'émet rien après un scan mobile | 🔴 aucun rafraîchissement live des entrées entre appareils |
| Vue opérationnelle | les stats `present` ne sont chargées qu'à l'ouverture d'une session, sans polling | 🔴 pas de dashboard global temps réel |
| Export email + présent/absent | `export-sessions` contient les emails et heures d'entrée ; les feuilles session ne contiennent que les personnes ayant un scan | 🟠 un absent n'apparaît pas et aucun statut explicite présent/absent n'est exporté |

Le second export `export-stats` est encore moins sûr pour ce besoin : certains calculs recherchent un
scan `IN` sans limiter le statut à `admitted`. Le livrable CDC doit donc s'appuyer sur
`export-sessions`, corrigé et testé, pas sur une cellule d'entrée vide interprétée manuellement.

### Contrat cible minimal

1. Après L9.1, PostgreSQL porte `present_count` et le met à jour dans la même transaction que le scan.
2. Un endpoint authentifié renvoie en **une réponse** les sessions de l'événement : salle, début/fin,
   `registered_count`, `present_count`, `unique_entered_count`, capacité physique effective,
   `fill_rate`, état et `updated_at`.
3. Après commit d'un `IN`, `OUT` ou undo, le serveur émet un événement agrégé dans une room
   authentifiée de l'organisation/événement. Aucun email ni identité n'est envoyé sur le socket.
4. Le dashboard reprend toujours un snapshot autoritaire au chargement et à la reconnexion. Le
   WebSocket accélère l'affichage mais ne devient jamais la source de vérité.
5. Pour le MVP BIL, un polling authentifié toutes les 3–5 secondes via le proxy PHP est acceptable si
   le JWT reste en session serveur. Ne pas exposer le JWT long-lived au navigateur uniquement pour le
   WebSocket. Un socket authentifié pourra être ajouté si le contrat sécurité est propre.

Pour LFD, une session représente le couple **salle + créneau** (`location`, `start_at`, `end_at`). Si
le client attend une agrégation de plusieurs sessions partageant une même salle/plage, il faut le
faire confirmer au call et définir les chevauchements ; ne pas sommer aveuglément des sessions qui
pourraient compter la même personne.

### Export post-event attendu

Pour chaque session, exporter au minimum : session, salle, début/fin, nom, prénom, **email**, statut
d'inscription session et statut de participation.

- `Présent` : au moins un `IN` au statut `admitted`, même si le dernier scan est `OUT` ;
- `Absent` : choix de session `confirmed`, mais aucun `IN` admis ;
- `Présent hors liste` : `IN` admis sans choix session, si la session autorise le walk-in ;
- `waitlisted`, `cancelled` et `blocked` restent identifiables mais ne doivent pas être étiquetés
  automatiquement « absent » ;
- conserver `first_in_at`, `last_out_at` et éventuellement le nombre de scans pour la preuve.

La feuille par session doit partir des **choix de session**, puis faire une jointure gauche sur les
scans admis. Partir uniquement des scans supprime mécaniquement les absents.

### Répartition sans nouveau chantier

| Lot | Responsable | Livrable |
| --- | --- | --- |
| H/L9.1 | vérité métier | compteur atomique, définition IN/OUT/doublon et capacité physique |
| J-ENTREES | lecture/live | snapshot event, calcul des métriques et propagation après commit |
| BIL | expérience client | dashboard salle/créneau, rôle/permissions et bouton d'export |
| T/K | preuve | E2E, concurrence puis répétition multi-scanner avec écran de contrôle |

### Tests de sortie

| Test | Preuve attendue |
| --- | --- |
| J-E1 | un premier `IN` admis augmente présence et visiteurs uniques ; `OUT` diminue seulement la présence ; doublon/rejet/replay ne sur-compte pas |
| J-E2 | deux scanners convergent vers le même snapshot serveur/dashboard, y compris après déconnexion/reconnexion |
| J-E3 | remplissage calculé avec `checkin_capacity_effective`, jamais `registered_count` |
| J-E4 | export : confirmé scanné `Présent`, confirmé non scanné `Absent`, participant ressorti toujours `Présent`, walk-in distingué |
| J-E5 | lecteur autorisé à voir les agrégats ; export accepté/refusé selon la décision RBAC du client |
| J-E6 | charge combinée avec dashboard actif, sans polling N+1 par session ni dégradation hors seuil |

### Estimation et ordre

Le lot brut représente **~2,5–5 j-dev**, mais L9.1, l'UI Sessions et une partie de l'export existent
déjà. L'incrément réellement nouveau est estimé à **~1,5–3 j-dev**, réparti dans J et BIL, sans
double-compter H/L9.1 :

1. terminer L9.1 et ses E2E ;
2. livrer le snapshot + rafraîchissement live J-ENTREES ;
3. brancher le dashboard et corriger l'export dans BIL ;
4. valider en répétition K, puis dans le k6 combiné final.
