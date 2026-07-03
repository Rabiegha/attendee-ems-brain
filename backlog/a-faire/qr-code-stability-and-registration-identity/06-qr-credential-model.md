# 06 — QrCredential Model (Conceptual)

> **STATUS : TO REVIEW / DISCOVERY**
>
> **Aucune migration Prisma. Aucun changement de schéma à ce stade.**
> Modèle conceptuel uniquement.

## 1. Entités proposées

### `QrCredential` (nouvelle)

| Champ | Type conceptuel | Description |
|---|---|---|
| `id` | UUID | Identifiant interne. |
| `token` | String (opaque, non devinable) | **Donnée encodée dans le QR**. Format à choisir en Phase 1 (UUIDv4 / base32 / NanoID). |
| `event_id` | UUID FK Event | Scope d'usage. Un QR appartient toujours à un événement précis. |
| `registration_id` | UUID FK Registration | **Nullable**. Peut devenir obsolète si la registration est supprimée puis recréée. |
| `stable_identity_key` | String | Clé d'identité stable copiée depuis la registration à l'émission (cf. [05-registration-identity-reconciliation.md](./05-registration-identity-reconciliation.md)). Permet la résolution de fallback. |
| `status` | Enum | `active`, `alias`, `revoked`, `blocked`, `expired`. |
| `resolution_mode` | Enum | `allow_check_in`, `warn_only`, `block_scan`, `supervisor_review`. |
| `issued_at` | DateTime | Horodatage d'émission. |
| `revoked_at` | DateTime nullable | Horodatage de révocation. |
| `replaced_by_id` | UUID nullable FK QrCredential | Pointeur vers le QR de remplacement. |
| `last_resolved_registration_id` | UUID nullable | Dernière registration résolue effectivement, pour debug/observabilité. |
| `metadata` | JSON | Contexte d'émission : email envoyé, badge imprimé, channel, version template. |
| `created_at` | DateTime | Standard. |
| `updated_at` | DateTime | Standard. |

### `Registration` (champs à ajouter conceptuellement)

| Champ | Type conceptuel | Description |
|---|---|---|
| `stable_identity_key` | String nullable | Clé d'identité dérivée à l'import (cf. [05-registration-identity-reconciliation.md](./05-registration-identity-reconciliation.md)). |
| `active_qr_credential_id` | UUID nullable FK QrCredential | QR actif courant (raccourci pour éviter une requête). |
| `qr_lifecycle_status` | Enum | `none`, `issued`, `sent`, `printed`, `scanned`, `revoked`. Permet les protections de suppression. |

> Les champs `status`, `attendee_id`, `event_id`, `id` existent déjà et ne sont pas modifiés ici.

## 2. Pourquoi `registration_id` peut être nullable ou devenir obsolète

C'est le point central du modèle :

- Si l'organisateur supprime la liste puis la réimporte, l'ancien `registration_id` n'existe plus en base.
- Le `QrCredential`, lui, **reste en base** (il n'est pas en cascade delete).
- Au scan suivant, le système :
  1. Trouve le `QrCredential` par `token`.
  2. Détecte que `registration_id` est invalide (ou que la registration n'existe plus).
  3. Tente une résolution via `event_id + stable_identity_key`.
  4. Si une registration actuelle existe pour cette clé, met à jour `last_resolved_registration_id` et autorise le check-in.

Le `QrCredential` joue alors le rôle d'un **passeport** qui survit aux remaniements administratifs.

## 3. Statuts possibles (`status`)

| Statut | Sens | Effet au scan |
|---|---|---|
| `active` | QR actuellement valide, attaché à une registration. | Tente résolution normale. |
| `alias` | Ancien QR conservé pour rétro-compatibilité, remplacé par un autre. | Résout vers la registration actuelle, peut afficher info « ancien QR ». |
| `revoked` | Volontairement coupé (compromission, erreur d'envoi). | Refus avec message clair. |
| `blocked` | Suspendu administrativement (fraude, abus). | Refus + alerte superviseur. |
| `expired` | Au-delà d'une date de validité optionnelle. | Refus avec message « expiré ». |

## 4. Modes de résolution possibles (`resolution_mode`)

| Mode | Sens |
|---|---|
| `allow_check_in` | Comportement standard : check-in autorisé si tout est OK. |
| `warn_only` | Check-in autorisé mais avec warning visible (utile pour les alias). |
| `block_scan` | Refus systématique au scan (override de `status`). |
| `supervisor_review` | Le scan ne décide pas seul : nécessite confirmation d'un superviseur. |

## 5. Règle de design fondamentale

> **Un ancien QR connu ne doit jamais être considéré invalide automatiquement.**

S'il est résolvable vers une registration **actuelle**, **valide**, dans le **bon événement**, il **doit être accepté**.

Les seuls cas de refus sont :

- `status = revoked` ou `blocked` ou `expired` ;
- `resolution_mode = block_scan` ;
- événement différent ;
- registration `cancelled` ;
- aucune résolution possible.

## 6. Relations conceptuelles

```
Event 1 ──< Registration 1 ──< QrCredential
                  │
                  └─── stable_identity_key
                                │
                                ▼
                       (résolution fallback)
                                │
QrCredential.stable_identity_key + event_id ───→ Registration actuelle
```

## 7. Volumétrie attendue

- 1 `QrCredential` par registration en V1.
- Multi-QR possible plus tard (badges multiples, événement multi-jour).
- Historique : on ne purge **jamais** les `QrCredential` `revoked` avant un délai produit (audit + RGPD).

## 8. Considérations non fonctionnelles

- Le `token` doit être **non devinable** (256 bits d'entropie minimum recommandé).
- Le `token` ne doit pas contenir de PII.
- Indexes recommandés : `(token)`, `(event_id, stable_identity_key)`, `(registration_id, status)`.
- Compatible avec une future migration vers une architecture event-driven : l'émission d'un `QrCredential` peut être déclenchée par un domaine `QrIssued` consommé par les modules email/badge (cf. workstream async-architecture).

## 9. Ce qui n'est PAS dans ce modèle (volontairement)

- Aucune logique de signature cryptographique (relève de l'Option E, hors scope V1).
- Aucune logique de TTL automatique (à discuter en Phase ultérieure).
- Aucune logique de rate limiting au scan (relève de la sécurité réseau).
- Aucun stockage de la donnée invité (nom, email) — reste sur `Registration`.

## 10. Mapping avec l'existant

| Existant | Cible conceptuelle | Action |
|---|---|---|
| QR encode `registration.id` | QR encode `QrCredential.token` | Phase 1 — émettre des tokens et migrer progressivement. |
| `external_id` (Registration) | `stable_identity_key` (Registration) | Évaluer : promotion de l'existant ou ajout d'un nouveau champ. À trancher avec produit. |
| Route publique `/qrcode/:registrationId` | Route publique `/qrcode/credential/:token` | Déprécier l'ancienne route après période de grace. |
| Aucun audit d'émission | `metadata` + `issued_at` | Nouveau. |
| Aucun statut QR | `status` + `resolution_mode` | Nouveau. |
