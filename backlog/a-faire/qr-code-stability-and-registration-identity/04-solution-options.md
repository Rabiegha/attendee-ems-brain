# 04 — Solution Options

> **STATUS : TO REVIEW / DISCOVERY**

Comparaison des options possibles pour résoudre le problème décrit dans [01-problem-statement.md](./01-problem-statement.md). Aucune option n'est tranchée à ce stade.

---

## Option A — `qrToken` stable sur `Registration`

### Principe

- Ajouter un champ `qr_token` (UUID v4 ou aléatoire) sur `Registration`.
- Le QR encode ce `qr_token` (et plus le `registration.id`).
- Le scan recherche la registration par `qr_token`.

### Avantages

- Simple à implémenter.
- N'expose plus l'UUID interne dans le QR public.
- Rapide à shipper en V1.
- Pas besoin de nouvelle table.

### Limites

- Si la registration est hard-deletée puis recréée, le token est perdu (même problème qu'aujourd'hui, juste avec un autre identifiant).
- Pas d'historique : impossible de tracer « ce QR a été émis le X, envoyé le Y, scanné le Z ».
- Révocation et remplacement difficiles (un seul token par registration).
- Pas de cycle de vie (`revoked`, `expired`, `replaced`).

### Verdict

Solution « pansement » : utile comme **prérequis** (couper l'exposition de l'UUID), insuffisante seule pour résoudre le problème.

---

## Option B — `QrCredential` lié à `Registration`

### Principe

- Nouvelle table `QrCredential` :
  - `id`, `token`, `event_id`, `registration_id`, `status`, `version`, `issued_at`, `revoked_at`.
- Le QR encode `token`.
- Le scan résout `QrCredential` → `Registration`.

### Avantages

- Historique : on conserve tous les QR émis.
- Révocation propre (`status = 'revoked'`).
- Remplacement traçable (`replacedById`).
- Plusieurs QR possibles par registration (utile pour les badges multiples, événements multi-jours).
- Audit complet : qui a émis, quand, pour qui.

### Limites

- **Ne résout pas le delete/recreate** : si `registration_id` est supprimé en cascade, le `QrCredential` perd son ancrage.
- Sans clé d'identité stable, impossible de retrouver la registration recréée.
- Plus complexe : nouvelle table, nouveau service, nouveau flow d'émission.

### Verdict

Indispensable mais insuffisant **seul**. Doit être combiné avec une `stableIdentityKey` (Option C).

---

## Option C — `QrCredential` + `stableIdentityKey`

### Principe

- `QrCredential` comme en Option B.
- En plus : `QrCredential` stocke `event_id` + `stable_identity_key`.
- `Registration` stocke aussi `stable_identity_key`.
- Index `(event_id, stable_identity_key)` sur Registration.
- Résolution au scan :
  1. Lookup `QrCredential.token`.
  2. Si `registration_id` toujours valide → utiliser.
  3. Sinon, fallback sur `event_id + stable_identity_key` pour retrouver la registration **actuelle**.

### Avantages

- Survit au delete/recreate.
- Permet « ancien QR encore acceptable » (use case 7).
- Compatible avec la réconciliation d'import (cf. [05-registration-identity-reconciliation.md](./05-registration-identity-reconciliation.md)).
- Permet la détection de cas ambigus (`needs_review`).
- Aligné avec les contraintes SaaS multi-tenant.

### Limites

- Nécessite de **définir la `stableIdentityKey`** : choix produit non trivial (cf. [05-registration-identity-reconciliation.md](./05-registration-identity-reconciliation.md)).
- Nécessite des **règles de matching** robustes (email normalisé, external_reference, employee_id).
- Nécessite la gestion des **conflits** (deux registrations matchent le même QR ? → `supervisor_review`).
- Complexité supérieure à A et B.

### Verdict

**Option recommandée provisoirement** (à valider produit) : c'est la seule qui adresse le scénario du delete/recreate.

---

## Option D — Interdire le hard delete après QR envoyé

### Principe

- Si un QR a été émis, envoyé ou imprimé, la registration ne peut **pas** être hard-deletée.
- Utiliser un statut `cancelled` / `inactive` / `archived` à la place.
- Les imports en mode « replace all » deviennent des `update + soft-cancel` plutôt que `delete + insert`.

### Avantages

- Résout le problème **à la racine** : pas de delete = pas de QR orphelin.
- Protège l'historique.
- Plus sûr opérationnellement.
- Conforme à l'esprit RGPD (soft delete + anonymisation contrôlée plutôt que hard delete impulsif).

### Limites

- Nécessite un changement UX : que se passe-t-il quand l'organisateur clique « Supprimer la liste » ?
- Nécessite de refondre la logique d'import pour qu'elle fasse de l'**upsert** systématique.
- Risque de surcharger la table avec des registrations « zombies ».
- Nécessite des opérations d'admin pour purger réellement (RGPD, demande explicite).

### Verdict

**Complément naturel** à l'Option C. Probablement à shipper en Phase 6 (cf. [11-future-implementation-plan-draft.md](./11-future-implementation-plan-draft.md)).

---

## Option E — QR signé (JWT/HMAC)

### Principe

- Le QR contient un payload signé : `{ eventId, registrationId, stableIdentityKey, exp }` + signature HMAC/JWT.
- Validation côté serveur par signature.

### Avantages

- Anti-falsification (impossible de forger un QR sans la clé).
- Le payload est self-contained (le scan peut en lire un sous-ensemble sans appel DB).
- Permet une vérification rapide offline (mobile en mode dégradé).

### Limites

- **Ne résout pas seul** le problème : il faut quand même résoudre vers une registration **actuelle** en base.
- Révocation difficile en mode stateless (nécessite une blacklist côté serveur, ce qui annule l'avantage).
- Rotation de clé compliquée (tous les anciens QR cassent ou demandent une grace period).
- Payload visible = légère fuite d'information (eventId, stableIdentityKey).

### Verdict

À considérer comme **complément futur** (Phase ultérieure, scan offline). Pas suffisant comme solution principale.

---

## Recommandation provisoire

Combiner :

1. **Couper l'exposition de l'UUID** : le QR public n'encode plus `registration.id` (issu de Option A).
2. **Introduire `QrCredential`** : table dédiée au cycle de vie du QR (Option B).
3. **Ajouter `stableIdentityKey`** sur `Registration` et `QrCredential` (Option C).
4. **Préserver les QR au réimport** : la réconciliation d'identité retrouve la registration (Option D, complément naturel).
5. **Encadrer le hard delete** : interdire ou exiger confirmation explicite si QR émis/envoyé/imprimé (Option D).
6. **Résolution de fallback** : `QrCredential.token` → `registration_id` direct → si KO, fallback `event_id + stable_identity_key`.
7. **(Plus tard)** Évaluer Option E pour scan offline en Phase ultérieure.

## Comparaison synthétique

| Critère | A (qrToken) | B (QrCredential) | C (B + stableKey) | D (no hard delete) | E (JWT signé) |
|---|---|---|---|---|---|
| Survit au delete/recreate | ❌ | ❌ | ✅ | ✅ | ❌ (sans C) |
| Historique QR | ❌ | ✅ | ✅ | ➖ | ❌ |
| Révocation propre | ❌ | ✅ | ✅ | ➖ | ⚠️ |
| Multi-QR par registration | ❌ | ✅ | ✅ | ➖ | ➖ |
| Anti-falsification | ❌ | ➖ | ➖ | ➖ | ✅ |
| Scan offline | ❌ | ❌ | ❌ | ❌ | ✅ |
| Complexité | 🟢 | 🟡 | 🟠 | 🟡 | 🟠 |
| Effort | S | M | L | M | L |

Légende : ✅ couvert · ❌ non couvert · ⚠️ partiel · ➖ neutre / hors sujet

## Décision attendue

Validation produit + tech sur le **combo C + D**, avec sequencing à définir dans [11-future-implementation-plan-draft.md](./11-future-implementation-plan-draft.md).
