# 03 — Current Risk Analysis

> **STATUS : TO REVIEW / DISCOVERY**
>
> Toute affirmation est **sourcée dans le code** ou marquée explicitement `NEEDS_CONFIRMATION`.

## Légende des statuts

- `FOUND_IN_CODE` : observation directe dans le code à la date de rédaction.
- `INFERRED_FROM_CODE` : déduction logique à partir de plusieurs éléments observés.
- `NOT_FOUND` : recherche effectuée, aucune occurrence trouvée.
- `NEEDS_CONFIRMATION` : nécessite validation produit/tech (interview, test).

Chemins relatifs au repo `attendee-ems-back/`.

---

## F1 — Le QR encode directement le `registration.id` (UUID)

**Statut** : `FOUND_IN_CODE`

**Sources** :

- `src/modules/registrations/registrations.service.ts`, méthode `generateQrCode()` (≈ ligne 3003) : `const qrData = registrationId; … QRCode.toBuffer(qrData, …)`. Le payload du QR est uniquement l'UUID de la registration.
- `src/modules/email/email.service.ts` (≈ ligne 244-256) : `QRCode.toBuffer(params.variables.registrationId, …)` — le QR inséré dans les emails encode directement le `registrationId`.
- `src/modules/email/qrcode.controller.ts` (≈ ligne 26) : route **publique non authentifiée** `GET /qrcode/:registrationId` qui retourne un PNG basé sur l'UUID passé en URL.
- `src/modules/badge-generation/badge-generation.service.ts` (≈ lignes 916, 1485) : `generateQRCode(registration.id)` — l'impression de badge encode également l'UUID.

**Risque** :

- Le QR est lié à un **identifiant technique mutable** : si la registration est supprimée puis recréée, le QR devient orphelin.
- L'UUID est aussi exposé dans une URL publique non authentifiée (`/qrcode/:registrationId`), ce qui en fait une fuite d'identifiant technique côté CDN/proxies.

**Impact** : élevé — c'est le cœur du problème décrit dans [01-problem-statement.md](./01-problem-statement.md).

**Question ouverte** : faut-il déprécier la route publique `/qrcode/:registrationId` au profit d'une route `/qrcode/credential/:token` ?

---

## F2 — Aucune entité `QrCredential` dédiée n'existe

**Statut** : `NOT_FOUND`

**Recherches effectuées** :

- `grep -r "QrCredential" src/` → aucun résultat.
- `grep "qr" prisma/schema.prisma` → unique occurrence : `qr_code_url String?` (ligne ≈ 820), qui semble être une **URL cachée** ou un champ de template, pas un credential.

**Risque** :

- Pas de cycle de vie du QR : impossible de marquer `revoked`, `replaced`, `expired`, `blocked`.
- Pas d'historique des QR émis pour une registration.
- Pas de moyen propre de réémettre un QR en gardant la trace de l'ancien.

**Impact** : élevé — bloque toute évolution propre (révocation, alias, supervisor_review).

---

## F3 — Aucun champ `stableIdentityKey` ou équivalent sur Registration

**Statut** : `NOT_FOUND` / `NEEDS_CONFIRMATION`

**Observations** :

- Champ `external_id` existe sur la table `registration` (utilisé à l'import lignes 1805-1846 de `registrations.service.ts` pour les colonnes `external_id`, `badge_id`, `qr_code`, `code`, `id externe`).
- Pas d'index combiné `(event_id, external_id)` confirmé visuellement — `NEEDS_CONFIRMATION` (lire `prisma/schema.prisma` pour valider).
- Pas de champ explicite type `stable_identity_key`, `employee_id`, `external_reference` distinct de `external_id`.

**Risque** :

- L'`external_id` est aujourd'hui surchargé : il sert à la fois de « QR code pré-imprimé », de « badge_id », et potentiellement de clé d'identité.
- Aucune sémantique claire « cette clé survit aux réimports ».

**Impact** : moyen — un champ existant pourrait être promu/clarifié, ou un nouveau champ ajouté.

---

## F4 — Le scan accepte UUID interne **ou** `external_id` scopé à l'event

**Statut** : `FOUND_IN_CODE`

**Source** : `src/modules/registrations/registrations.service.ts` (≈ ligne 3056-3094) — commentaire explicite :
> « Check-in via scan code (UUID interne OU external_id). Résout d'abord par UUID, puis par external_id scopé à l'event. »

**Risque** :

- Bonne nouvelle : il existe déjà un fallback `external_id` scopé event.
- Mauvaise nouvelle : ce fallback ne se déclenche **que si l'`external_id` a été fourni à l'import**. Sans `external_id`, la résolution se limite à l'UUID, qui peut avoir disparu.

**Impact** : moyen — base existante intéressante pour bâtir la résolution via `stableIdentityKey`.

---

## F5 — Hard delete possible sur Registration

**Statut** : `FOUND_IN_CODE`

**Sources** :

- `src/modules/registrations/registrations.service.ts` (≈ ligne 1440) : `permanentDelete(id, orgId)` → `prisma.registration.delete({ where: { id } })`.
- `src/modules/registrations/registrations.service.ts` (≈ ligne 2177) : `bulkDelete(ids, orgId)`.
- `src/modules/registrations/registrations.service.ts` (≈ ligne 2926) : `prisma.badge.deleteMany(…)` — les badges sont aussi hard-deleted.

**Risque** :

- Une registration peut être supprimée alors même qu'un QR a été émis et envoyé.
- Aucune protection conditionnelle visible (`if qr_sent_at then forbid hard delete`).

**Impact** : élevé — cause racine directe du scénario de bascule QR orphelin.

**Question ouverte** : existe-t-il un champ `qr_sent_at`, `qr_printed_at`, `qr_emailed_at` ? `NEEDS_CONFIRMATION`.

---

## F6 — Pas de soft delete visible pour Registration

**Statut** : `INFERRED_FROM_CODE` / `NEEDS_CONFIRMATION`

**Observations** :

- Aucune occurrence évidente de `deleted_at`, `is_active = false`, ou `status = 'archived'` utilisée comme alternative au hard delete dans le flow de suppression.
- La méthode `permanentDelete` confirme l'intention de hard-deleter.

**Risque** :

- Une fois supprimée, la registration est définitivement perdue avec son éventuel QR associé (statut, historique).

**Impact** : élevé.

---

## F7 — Import : delete-then-recreate vs upsert

**Statut** : `NEEDS_CONFIRMATION`

**Observations** :

- L'import gère du **upsert partiel** : sur match `external_id` ou email, certaines colonnes sont mises à jour (cf. lignes 1980-2090 de `registrations.service.ts`).
- Il existe `deleteMany` sur `RegistrationTableChoice` et `RegistrationSessionChoice` (lignes 1998, 2005, 2053, 2060) — ce sont les **choix** (tables/sessions) qui sont nettoyés, **pas la registration elle-même** dans ce flow.
- Le comportement en cas de « ligne absente du nouveau fichier » n'est pas évident dans le code lu. `NEEDS_CONFIRMATION` :
  - L'import marque-t-il les rows manquants comme `cancelled` ?
  - Les laisse-t-il intacts ?
  - Existe-t-il un mode « replace all » qui hard-delete avant import ?

**Risque** :

- Si un mode « replace all » existe, c'est la voie royale vers le scénario QR orphelin.
- Si l'organisateur fait manuellement « tout supprimer puis réimporter » via l'UI, on est dans le scénario F5 + F2.

**Impact** : élevé si confirmé.

---

## F8 — Email : envoi du QR sans persister la trace d'émission

**Statut** : `INFERRED_FROM_CODE` / `NEEDS_CONFIRMATION`

**Source** : `src/modules/email/email.service.ts` (≈ lignes 244-256) — le QR est généré à la volée puis attaché à l'email. Aucune écriture en base type `qr_sent_at = NOW()` n'est visible dans le snippet observé.

**Risque** :

- Impossible de savoir a posteriori si un QR a effectivement été envoyé à l'invité.
- Empêche toute règle « interdire hard delete si QR envoyé ».
- Empêche un audit type « combien d'invités ont reçu leur QR ? ».

**Impact** : élevé pour la gouvernance produit, moyen pour le risque immédiat.

---

## F9 — Badge : impression sans trace explicite « QR imprimé »

**Statut** : `INFERRED_FROM_CODE` / `NEEDS_CONFIRMATION`

**Source** : `src/modules/badge-generation/badge-generation.service.ts` — `generateQRCode(registration.id)` est appelé lors de la génération de badge (lignes 916 et 1485), mais aucun champ `badge_printed_at` ou `qr_printed_at` n'a été observé.

**Risque** : identique à F8 — pas de trace permettant de protéger un QR « déjà sorti » du système.

**Impact** : moyen.

---

## F10 — Aucune notion de « version » ou d'« alias » pour les QR

**Statut** : `NOT_FOUND`

**Risque** :

- Impossible d'émettre un nouveau QR en gardant l'ancien valide en mode `alias` (use case 7).
- Impossible de tracer un remplacement (`replacedById`).

**Impact** : moyen.

---

## Synthèse risques

| ID | Finding | Statut | Impact |
|---|---|---|---|
| F1 | QR = `registration.id` brut | FOUND | Élevé |
| F2 | Pas de `QrCredential` | NOT_FOUND | Élevé |
| F3 | Pas de `stableIdentityKey` claire | NEEDS_CONFIRMATION | Moyen |
| F4 | Scan UUID + external_id scopé event | FOUND | Moyen (base utile) |
| F5 | Hard delete autorisé sans condition | FOUND | Élevé |
| F6 | Pas de soft delete Registration | INFERRED | Élevé |
| F7 | Comportement reimport delete/recreate | NEEDS_CONFIRMATION | Élevé si confirmé |
| F8 | Pas de `qr_sent_at` persisté | INFERRED | Moyen |
| F9 | Pas de `qr_printed_at` persisté | INFERRED | Moyen |
| F10 | Pas de versioning / alias QR | NOT_FOUND | Moyen |

## Prochaines vérifications recommandées (avant Phase 0)

1. Lire `prisma/schema.prisma` pour confirmer F3, F6 (champ `deleted_at` ?), index `(event_id, external_id)`.
2. Tracer le flow d'import complet pour confirmer F7 (mode `replace all` ?).
3. Vérifier les hooks d'envoi email pour confirmer F8 (existe-t-il un log d'audit ?).
4. Vérifier le module badge pour confirmer F9.
5. Faire un grep `prisma.registration.delete` pour mesurer la surface d'attaque de F5.
