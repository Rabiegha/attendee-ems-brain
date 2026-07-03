# 07 — Scan Behavior

> **STATUS : TO REVIEW / DISCOVERY**

## 1. Objectif

Définir le **comportement cible** du scan de QR le jour J, de façon à ce que :

- l'hôtesse ait toujours une **action exploitable** ;
- les cas ambigus ne bloquent jamais silencieusement un invité légitime ;
- la sécurité (révocation, blocage) reste stricte et lisible ;
- la rétro-compatibilité avec les anciens QR est garantie.

## 2. Flux cible — pseudo-code

```
Entrée :
  - scanned_token : string (contenu du QR scanné)
  - context_event_id : UUID (événement courant côté mobile/web)
  - context_org_id : UUID

Sortie :
  - { result, registration?, qrCredential?, message_for_host, log_level }

Algorithme :

1. Si scanned_token introuvable en table QrCredential :
     → vérifier si scanned_token correspond à un Registration.id (rétro-compat anciens QR)
     → si oui : router vers le flux registration directe
     → sinon : result = 'unknown_token'

2. Sinon : qrCredential trouvé
     2.1 Si qrCredential.event_id ≠ context_event_id :
           → result = 'event_mismatch'

     2.2 Si qrCredential.status ∈ { revoked, blocked, expired } :
           → result = qrCredential.status

     2.3 Si qrCredential.resolution_mode = block_scan :
           → result = 'blocked'

     2.4 Si qrCredential.registration_id existe :
           Chercher registration par id :
             - existe et non-cancelled → continuer en 3
             - existe et cancelled → result = 'cancelled'
             - n'existe plus → tomber en 2.5 (fallback)

     2.5 Fallback résolution via stable_identity_key :
           candidats = registrations WHERE event_id = context_event_id
                       AND stable_identity_key = qrCredential.stable_identity_key
                       AND status NOT IN (cancelled)
           - 0 candidat → result = 'unresolved_token'
           - 1 candidat → continuer en 3, marquer alias_resolved
           - >1 candidat → result = 'supervisor_review'

3. Décider du check-in :
     - Si déjà checked-in → result = 'already_checked_in'
     - Si resolution_mode = supervisor_review → result = 'supervisor_review'
     - Si resolution_mode = warn_only → result = 'valid_with_warning'
     - Sinon → result = 'valid' (ou 'alias_resolved' si via fallback)

4. Persister :
     - log scan (event_id, qr_credential_id, registration_id, result, user_id, timestamp)
     - mettre à jour qrCredential.last_resolved_registration_id si différent
     - mettre à jour qr_lifecycle_status = 'scanned' sur la registration

5. Retourner réponse exploitable au mobile.
```

## 3. Résultats possibles (`result`)

| Code | Sens | Action hôtesse |
|---|---|---|
| `valid` | Tout est OK, premier scan. | Laisser entrer. |
| `valid_with_warning` | OK avec mention (ex : type participant changé). | Laisser entrer, lire le warning. |
| `already_checked_in` | Déjà checké à HH:MM. | Vérifier (entrée multiple ?). |
| `alias_resolved` | Ancien QR mais reconnu via stableIdentityKey. | Laisser entrer normalement. |
| `supervisor_review` | Ambigüité, plusieurs registrations possibles. | Appeler superviseur. |
| `revoked` | QR explicitement révoqué. | Refuser, expliquer. |
| `blocked` | QR suspendu administrativement. | Refuser, alerter superviseur. |
| `expired` | Au-delà de la date de validité. | Refuser, rediriger vers accueil. |
| `cancelled` | Inscription annulée. | Refuser, expliquer. |
| `event_mismatch` | QR d'un autre événement. | Refuser, rediriger. |
| `unknown_token` | QR jamais émis par notre système. | Refuser, vérifier manuellement. |
| `unresolved_token` | QR connu mais aucune registration retrouvable. | Recherche manuelle assistée. |

## 4. Exemples de messages UX (cible)

| `result` | Message hôtesse (FR) |
|---|---|
| `valid` | « ✅ Marie Dupont — Premium. Check-in autorisé. » |
| `valid_with_warning` | « ✅ Marie Dupont — Standard *(type modifié récemment)*. Check-in autorisé. » |
| `already_checked_in` | « ⚠️ Déjà checké à 09:32. Confirmer une seconde entrée ? » |
| `alias_resolved` | « ✅ Ancien QR reconnu. Marie Dupont — Premium. Check-in autorisé. » |
| `supervisor_review` | « ⚠️ QR reconnu mais correspondance ambiguë (2 inscriptions au même nom). Appeler superviseur. » |
| `revoked` | « ❌ Ce QR a été révoqué. Voir l'accueil pour un QR de remplacement. » |
| `blocked` | « ❌ Ce QR est bloqué. Contacter un superviseur. » |
| `expired` | « ❌ Ce QR a expiré. Rediriger vers l'accueil. » |
| `cancelled` | « ❌ Cette inscription a été annulée. » |
| `event_mismatch` | « ❌ Ce QR concerne un autre événement. » |
| `unknown_token` | « ❌ QR inconnu. Rechercher l'invité par nom. » |
| `unresolved_token` | « ⚠️ Ce QR ne retrouve plus d'inscription. Rechercher par nom. » |

> Les messages techniques (codes erreur, stack traces, UUID) ne doivent **jamais** apparaître dans l'UI hôtesse.

## 5. Réponse API recommandée (structure)

```json
{
  "result": "alias_resolved",
  "registration": {
    "id": "uuid",
    "first_name": "Marie",
    "last_name": "Dupont",
    "company": "Acme",
    "attendee_type": "Premium",
    "checked_in_at": null
  },
  "warning": "Reconnu via clé d'identité stable (registration recréée après réimport).",
  "message": "Ancien QR reconnu. Marie Dupont — Premium. Check-in autorisé.",
  "next_action": "allow_check_in",
  "supervisor_required": false,
  "log_id": "uuid"
}
```

## 6. Principes de design UX

1. **Toujours fournir une action.** Même `unknown_token` propose « Rechercher par nom ».
2. **Distinguer refus durs et ambigus.** Un `supervisor_review` ne doit pas ressembler à un `revoked`.
3. **Pas de jargon technique.** « Token », « UUID », « stale » ne doivent pas apparaître côté hôtesse.
4. **Mention discrète des warnings.** Un `alias_resolved` doit être visible mais ne pas inquiéter l'invité.
5. **Audit complet côté backend.** Tout scan est loggé pour analyse a posteriori (KPI, support, sécurité).

## 7. Rétro-compatibilité (anciens QR encodant `registration.id`)

Pendant la période de grace :

- Si `scanned_token` matche un `Registration.id` existant : router vers une résolution directe et logger comme `legacy_token`.
- Si `scanned_token` matche un `Registration.id` qui n'existe plus : tenter une résolution via `event_id + name + email` (best effort) → `supervisor_review` en cas de doute.
- Émettre un **nouveau** `QrCredential` lors d'opérations futures (régénération badge, renvoi email).
- Période de grace recommandée : la durée de vie du plus long événement actif + marge. À discuter produit.

## 8. Mobile / Print client

- Le **mobile** appelle l'API backend pour décider du résultat. Aucun arbitrage côté client.
- Le **print client** ne décide pas du check-in : il imprime ce que le backend lui retourne.
- En cas de **scan offline** futur (cf. Option E), prévoir un mode dégradé où le mobile accepte conditionnellement et synchronise au retour réseau. **Hors scope V1.**

## 9. Anti-patterns à proscrire

- Réponse muette `{ "error": "not_found" }` sans contexte exploitable.
- Bouton « Forcer le check-in » sans audit ni rôle.
- Affichage du `token` dans l'UI.
- Décision de validité prise côté mobile à partir du payload QR.
- Cache local de validité QR de plus de quelques secondes.

Voir [08-things-to-avoid.md](./08-things-to-avoid.md) pour la liste complète.
