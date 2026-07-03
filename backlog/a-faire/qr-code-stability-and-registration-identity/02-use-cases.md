# 02 — Use Cases

> **STATUS : TO REVIEW / DISCOVERY**

Ces use cases définissent **ce que la solution cible doit savoir gérer** sans préjuger de l'implémentation. Chaque cas doit être traçable dans [10-acceptance-criteria-draft.md](./10-acceptance-criteria-draft.md).

---

## Use case 1 — Update simple après envoi QR

**Contexte** : le QR a été envoyé par email à l'invité. L'organisateur corrige ensuite le nom, l'entreprise ou le type de participant.

**Attendu** :

- Le QR existant **continue de fonctionner**.
- Aucune régénération de QR n'est nécessaire.
- Les nouvelles données sont reflétées au scan (badge à jour).

**Anti-pattern** : régénérer un nouveau QR à chaque update et invalider l'ancien sans notification.

---

## Use case 2 — Réimport avec même email

**Contexte** : l'organisateur réimporte la liste. La ligne pour l'invité a le **même email** (clé fonctionnelle la plus courante).

**Attendu** :

- Le système **retrouve la registration existante** (par `eventId + normalizedEmail`).
- Le QR émis reste valide.
- Aucune duplication n'est créée.
- Les éventuelles colonnes mises à jour (titre, entreprise) sont appliquées.

**Anti-pattern** : créer une nouvelle registration parce que l'algorithme d'import fait `delete-then-recreate` aveugle.

---

## Use case 3 — Réimport après suppression/recréation

**Contexte** : l'ancienne registration a été **supprimée** (hard delete). L'organisateur réimporte, ce qui **crée une nouvelle** registration pour le même invité.

**Attendu** :

- Le `QrCredential` existant peut être **résolu vers la nouvelle registration** via `eventId + stableIdentityKey`.
- Le scan autorise le check-in (éventuellement avec un warning interne).
- L'historique du `QrCredential` est préservé.

**Anti-pattern** : retourner « QR inconnu » au jour J alors que l'invité est bien présent dans la liste actuelle.

---

## Use case 4 — Changement d'email

**Contexte** : entre deux imports, l'email d'un invité change (changement d'entreprise, faute de frappe corrigée).

**Attendu** :

- Si une `external_reference` ou `employee_id` est fournie, le matching se fait sur cette clé : la même personne est retrouvée, le QR continue de fonctionner.
- Sinon, le système **ne casse pas silencieusement** : il marque le cas en `needs_review` plutôt que de créer une nouvelle registration et un nouveau QR.
- L'organisateur reçoit un **rapport de conflits** post-import.

**Anti-pattern** : supprimer l'ancien et créer un nouveau sans avertir, faisant perdre l'historique et invalidant le QR envoyé.

---

## Use case 5 — Doublons dans l'import

**Contexte** : le fichier d'import contient **deux lignes avec le même email** ou la même `external_reference`.

**Attendu** :

- L'import **détecte le doublon** avant écriture.
- Le système **ne génère pas deux QR actifs concurrents** sans décision explicite.
- Le doublon est remonté dans le rapport d'import.

**Anti-pattern** : créer deux registrations, deux QR, et laisser l'utilisateur découvrir le problème au jour J.

---

## Use case 6 — QR explicitement révoqué

**Contexte** : un QR a été compromis (capture d'écran circulant, mauvaise personne destinataire). L'organisateur le révoque.

**Attendu** :

- Le statut du `QrCredential` passe à `revoked`.
- Le scan retourne un message clair : « Ce QR a été révoqué ».
- Un QR de remplacement peut être émis (lien `replacedById`).
- L'historique reste consultable.

**Anti-pattern** : supprimer le QR de la base et perdre la trace.

---

## Use case 7 — Ancien QR encore acceptable

**Contexte** : l'invité présente un ancien QR (généré dans une version antérieure). Le QR pointe vers une registration qui existe toujours, sans révocation.

**Attendu** :

- Le scan **autorise le check-in**.
- Éventuellement, log un warning interne `alias_resolved` pour observabilité.
- L'hôtesse ne voit aucun message technique anxiogène.

**Anti-pattern** : afficher « QR obsolète, veuillez contacter le support » à un invité légitime.

---

## Use case 8 — Registration cancelled

**Contexte** : l'inscription a été annulée (désistement, exclusion).

**Attendu** :

- Le scan retourne un message clair : « Inscription annulée ».
- L'historique du `QrCredential` reste consultable.
- Pas de check-in possible sans override superviseur.

**Anti-pattern** : silencieusement valider le check-in d'une personne annulée.

---

## Use case 9 — Event mismatch

**Contexte** : l'invité présente un QR d'un autre événement (ou ancien).

**Attendu** :

- Le scan détecte l'`eventId` du QR ≠ `eventId` du contexte de scan.
- Message clair : « Ce QR ne correspond pas à cet événement ».

**Anti-pattern** : crash, message technique, ou pire : check-in croisé entre événements.

---

## Use case 10 — Jour J / hôtesse

**Contexte** : à l'entrée, l'hôtesse a une dizaine de secondes par scan et doit pouvoir prendre une décision même quand le système hésite.

**Attendu** :

- Toute réponse de scan est **exploitable** : pas de message uniquement technique (`E_UNRESOLVED_TOKEN_42`).
- Les cas ambigus déclenchent un mode `supervisor_review` clair.
- Si la personne est légitime, le scan **doit l'aider à la retrouver**, pas la bloquer en silence.
- Le mobile doit afficher : nom, statut, action recommandée.

**Anti-pattern** : un message « QR inconnu » sans alternative, qui force l'invité à attendre 10 minutes pendant que l'hôtesse cherche manuellement.

---

## Synthèse

| # | Cas | Attendu principal | Risque si non traité |
|---|---|---|---|
| 1 | Update après envoi | QR conservé | Régénération inutile, confusion invité |
| 2 | Réimport même email | Match par email, QR conservé | Doublon, QR cassé |
| 3 | Delete + recreate | Résolution via stableIdentityKey | QR orphelin au jour J |
| 4 | Email changé | Match par external_ref ou needs_review | Casse silencieuse |
| 5 | Doublons import | Détection avant écriture | Deux QR actifs, ambigu |
| 6 | Révocation | Statut `revoked` clair | Pas de moyen propre de couper un QR |
| 7 | Ancien QR valide | Alias resolved | Refus injustifié |
| 8 | Cancelled | Message clair | Check-in fantôme |
| 9 | Event mismatch | Refus clair | Check-in croisé |
| 10 | UX hôtesse | Action exploitable | Blocage opérationnel jour J |
