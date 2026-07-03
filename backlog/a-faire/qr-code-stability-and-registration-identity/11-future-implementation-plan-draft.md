# 11 — Future Implementation Plan (Draft)

> **STATUS : TO REVIEW / DISCOVERY**
>
> **Ne pas implémenter maintenant.** Ce plan documente un séquencement possible une fois la phase de discovery validée.
>
> Toute affirmation sur le code existant doit être sourcée ou marquée `NEEDS_CONFIRMATION` (voir [03-current-risk-analysis.md](./03-current-risk-analysis.md)).

## Principes directeurs

1. **Pas de big-bang.** Chaque phase est livrable indépendamment.
2. **Rétro-compatibilité d'abord.** Les anciens QR doivent continuer de fonctionner pendant la période de grace.
3. **La DB reste source de vérité.** Pas de Redis ni de stateless qui décide seul.
4. **Compatibilité async/event-driven.** Toute émission de QR est idempotente et publishable comme événement domaine.
5. **Aucune phase ne démarre tant que la précédente n'est pas validée en prod.**

---

## Phase 0 — Audit

**Objectif** : confirmer toutes les hypothèses de [03-current-risk-analysis.md](./03-current-risk-analysis.md).

### Livrables

- Document de confirmation pour chaque finding (F1 à F10).
- Cartographie complète des points d'émission de QR :
  - email (`email.service.ts`),
  - badge (`badge-generation.service.ts`),
  - route publique (`email/qrcode.controller.ts`),
  - controller `registrations/:id/qr-code`.
- Cartographie des endpoints de scan et de check-in.
- Cartographie des flows d'import (en particulier mode `replace all` éventuel).
- Cartographie des points de hard delete (`prisma.registration.delete`, `bulkDelete`, `permanentDelete`).
- Audit produit : combien de QR sont déjà envoyés à des utilisateurs réels ? (question Q2)

### Acteurs

- 1 dev backend (lecture code).
- 1 PM produit (interviews clients).

### Critère de sortie

- Toutes les questions Audit de [09-open-questions.md](./09-open-questions.md) sont passées de `OPEN` à `DECIDED`.

---

## Phase 1 — Stabiliser le QR public (sans changement de modèle)

**Objectif** : couper l'exposition de l'UUID interne dans le QR sans encore introduire `QrCredential`.

### Livrables

- Introduction d'un champ `qr_token` (UUID v4 ou NanoID) sur `Registration` — uniquement si validé en Q19.
- Le QR encode désormais `qr_token` (et non plus `registration.id`).
- Backfill : génération d'un `qr_token` pour toutes les registrations existantes.
- Nouvelle route publique `/qrcode/token/:token` qui retourne le PNG.
- Ancienne route `/qrcode/:registrationId` conservée en grace, mais loggée comme `legacy`.
- Scan accepte les deux formats pendant la période de grace.

### Risques

- Migration de tous les emails déjà envoyés : non, ils restent valides via rétro-compat.
- Performance : indexer `qr_token` UNIQUE.

### Critère de sortie

- Tous les nouveaux QR encodent `qr_token`.
- L'UUID interne n'apparaît plus dans les nouveaux emails / badges.

---

## Phase 2 — Ajouter `stable_identity_key` sur Registration

**Objectif** : préparer la résolution de fallback.

### Livrables

- Ajout du champ `stable_identity_key` sur `Registration` (string, nullable).
- Définition produit/tech de la **règle de dérivation** : `external_reference` ?? `employee_id` ?? `normalized_email`.
- Stockage de la **source effective** (`stable_identity_source` enum) si validé en Q20.
- Backfill : application de la règle sur les registrations existantes.
- Rapport de backfill : combien de registrations ont une clé fiable vs fallback email vs aucune.
- Index unique partiel `(event_id, stable_identity_key) WHERE stable_identity_key IS NOT NULL`.
- Gestion des conflits du backfill : marquer `needs_review` les cas où deux registrations matchent la même clé.

### Critère de sortie

- 100 % des registrations actives ont une `stable_identity_key` ou sont marquées explicitement « sans clé fiable ».

---

## Phase 3 — Introduire `QrCredential`

**Objectif** : modèle conceptuel cible décrit dans [06-qr-credential-model.md](./06-qr-credential-model.md).

### Livrables

- Table `QrCredential` avec les champs : `id`, `token`, `event_id`, `registration_id`, `stable_identity_key`, `status`, `resolution_mode`, `issued_at`, `revoked_at`, `replaced_by_id`, `last_resolved_registration_id`, `metadata`, timestamps.
- Service `QrCredentialService` : émission, révocation, remplacement, lookup.
- Migration des `qr_token` (Phase 1) vers des `QrCredential` :
  - chaque `qr_token` devient un `QrCredential` `active`,
  - le champ `qr_token` sur Registration devient un raccourci (`active_qr_credential_id`).
- Hooks d'émission depuis email et badge : enregistrement explicite (`metadata.channel = 'email' | 'badge'`).
- Champ `qr_lifecycle_status` sur Registration mis à jour à chaque étape.

### Critère de sortie

- Tout nouvel envoi de QR crée un `QrCredential`.
- L'historique d'émission est consultable.

---

## Phase 4 — Modifier le scan

**Objectif** : flux de scan décrit dans [07-scan-behavior.md](./07-scan-behavior.md).

### Livrables

- Endpoint de scan refondu : lookup par `token` → résolution directe → fallback `stable_identity_key`.
- Gestion des résultats : `valid`, `valid_with_warning`, `alias_resolved`, `already_checked_in`, `supervisor_review`, `revoked`, `blocked`, `expired`, `cancelled`, `event_mismatch`, `unknown_token`, `unresolved_token`.
- Réponse API structurée (cf. exemple JSON dans [07-scan-behavior.md](./07-scan-behavior.md)).
- Logging systématique des scans (audit).
- Rétro-compatibilité : si le payload est un UUID `registration.id` (ancien QR), résolution best-effort.

### Mobile / front

- Adaptation des écrans pour afficher : message, action recommandée, bouton « Recherche manuelle », bouton « Appeler superviseur ».
- Aucun arbitrage côté mobile : tout vient du backend.

### Critère de sortie

- Tests e2e des 12 codes de résultat passent.
- Tests de scénario jour J validés (import → envoi → modif → réimport → scan).

---

## Phase 5 — Modifier les imports (réconciliation d'identité)

**Objectif** : règles de [05-registration-identity-reconciliation.md](./05-registration-identity-reconciliation.md).

### Livrables

- Algorithme d'upsert hiérarchique : `external_reference` → `employee_id` → `normalized_email` → `normalized_phone` → fuzzy.
- Détection des doublons avant écriture.
- Marquage `needs_review` pour les cas ambigus.
- Préservation des QR existants : aucun nouveau `QrCredential` n'est émis si la registration est reconnue.
- Rapport d'import : matchés, créés, conflits, needs_review, absents du nouvel import.
- Suppression du chemin « delete-all + recreate » (s'il existe — à confirmer en Phase 0 / F7).

### Critère de sortie

- Tests d'intégration import couvrent tous les cas de [02-use-cases.md](./02-use-cases.md).
- Aucun cas d'import ne génère un QR orphelin.

---

## Phase 6 — Protection contre la suppression

**Objectif** : règles d'Option D dans [04-solution-options.md](./04-solution-options.md).

### Livrables

- Garde : `prisma.registration.delete` bloqué si `qr_lifecycle_status ∈ { issued, sent, printed, scanned }`.
- Endpoint `cancel` : statut `cancelled` + révocation `QrCredential` proprement.
- Endpoint `archive` : statut `archived` + `QrCredential` conservé en consultation.
- Endpoint `force_delete` (rôle admin uniquement, raison obligatoire, audit) pour les besoins RGPD.
- Adaptation de l'UI : remplacer « Supprimer » par « Annuler » (cas standard) avec option « Suppression définitive » (cas admin).
- Audit log de toutes les opérations destructives.

### Critère de sortie

- Plus aucune registration avec QR `sent` ou `printed` ne peut être hard-deletée par accident.

---

## Phase 7 — Tests, observabilité, runbook

**Objectif** : industrialiser et opérationnaliser.

### Livrables

- Suite de tests unitaires + intégration + e2e couvrant 100 % de [10-acceptance-criteria-draft.md](./10-acceptance-criteria-draft.md).
- Tests de migration : anciens QR pendant la période de grace.
- Tests de scénario jour J automatisés.
- Dashboard d'observabilité : émission, envoi, impression, scans, résultats, alias_resolved, needs_review.
- Alertes : taux anormal de `unresolved_token`, `supervisor_review`, `unknown_token`.
- Runbook support : « Un invité dit que son QR ne marche pas, quoi faire en 30 secondes ? ».
- Documentation client : « Comment fournir une `external_reference` fiable ».

### Critère de sortie

- Taux de scan `valid` + `alias_resolved` ≥ 99,5 % en pré-prod.
- Aucun ticket support de type « QR cassé après réimport » sur 1 mois.

---

## Contraintes globales

- **Ne pas implémenter maintenant.**
- Rester en backlog discovery jusqu'à validation produit + tech.
- Toute affirmation sur le code doit être sourcée ou marquée `NEEDS_CONFIRMATION`.
- Préparer les futures tâches mais ne pas les exécuter.
- Garder la DB comme source de vérité.
- Garder la solution compatible avec BullMQ/Redis et future event-driven (cf. workstream [async-architecture](../../workstreams/async-architecture/README.md)).
- Toute évolution de schéma fait l'objet d'une ADR séparée avant migration.

## Dépendances entre phases

```
Phase 0 (Audit)
   │
   ▼
Phase 1 (qr_token)           Phase 2 (stable_identity_key)
   │                                │
   └─────────────┬──────────────────┘
                 ▼
           Phase 3 (QrCredential)
                 │
        ┌────────┴────────┐
        ▼                 ▼
  Phase 4 (Scan)     Phase 5 (Imports)
        │                 │
        └────────┬────────┘
                 ▼
          Phase 6 (Suppression)
                 │
                 ▼
          Phase 7 (Tests + Obs)
```

Phases 1 et 2 peuvent être menées en parallèle.
Phases 4 et 5 peuvent être menées en parallèle après Phase 3.

## Estimation préliminaire (à valider)

| Phase | Effort | Risque | Bloquant pour V1 ? |
|---|---|---|---|
| 0 | S | Faible | Oui |
| 1 | M | Faible | Oui |
| 2 | M | Moyen (backfill) | Oui |
| 3 | L | Moyen | Oui |
| 4 | L | Moyen | Oui |
| 5 | L | Élevé (refonte import) | Oui |
| 6 | M | Moyen (UX) | Oui |
| 7 | M | Faible | Oui |

> Tous les efforts sont indicatifs. Aucun engagement de calendrier à ce stade.
