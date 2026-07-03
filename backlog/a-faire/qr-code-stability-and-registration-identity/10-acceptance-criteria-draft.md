# 10 — Acceptance Criteria (Draft)

> **STATUS : TO REVIEW / DISCOVERY**
>
> Critères provisoires destinés à guider la définition de la V1 et à valider l'implémentation future. Aucun code à écrire à ce stade.

## A. Stabilité du QR

- [ ] **AC-A1** Un QR émis et envoyé continue de fonctionner après modification simple des données de la registration (nom, entreprise, type participant).
- [ ] **AC-A2** Un QR émis et envoyé continue de fonctionner après réimport si l'identité stable est retrouvée.
- [ ] **AC-A3** Un QR émis et envoyé continue de fonctionner après suppression + recréation de la registration, si `stable_identity_key` permet la résolution.
- [ ] **AC-A4** Le scan ne dépend plus uniquement de `attendee.id`.
- [ ] **AC-A5** Le scan ne dépend pas uniquement de `registration.id` sans mécanisme de fallback.

## B. Résolution d'identité

- [ ] **AC-B1** Un QR connu peut être résolu via `event_id + stable_identity_key` si `registration_id` n'est plus valide.
- [ ] **AC-B2** Une résolution de fallback est tracée (`last_resolved_registration_id` mis à jour, log d'audit).
- [ ] **AC-B3** Une ambiguïté (≥ 2 registrations possibles) déclenche `supervisor_review` et n'arbitre jamais seule.
- [ ] **AC-B4** L'absence totale de résolution retourne `unresolved_token` avec piste d'action côté hôtesse.

## C. Cycle de vie du QR

- [ ] **AC-C1** Le statut d'un QR peut prendre les valeurs `active`, `alias`, `revoked`, `blocked`, `expired`.
- [ ] **AC-C2** Un QR `revoked` ou `blocked` est refusé clairement au scan (message exploitable).
- [ ] **AC-C3** Un QR de remplacement référence son prédécesseur (`replaced_by_id`).
- [ ] **AC-C4** L'historique des QR n'est jamais hard-deleted (audit + RGPD).

## D. Import et réconciliation

- [ ] **AC-D1** Un réimport ne régénère pas un QR si la registration est reconnue.
- [ ] **AC-D2** Un réimport avec même email match une registration existante (par `event_id + normalized_email` ou `external_reference`).
- [ ] **AC-D3** Les doublons dans l'import sont détectés avant écriture et marqués `needs_review`.
- [ ] **AC-D4** Un import qui produit des `needs_review` génère un rapport visible côté organisateur.
- [ ] **AC-D5** Une registration absente du nouvel import n'est jamais hard-deletée automatiquement.

## E. Suppression sécurisée

- [ ] **AC-E1** Une registration avec QR `issued`, `sent`, `printed` ou `scanned` ne peut pas être hard-deletée sans action explicite (`force_delete` avec rôle adapté + raison).
- [ ] **AC-E2** L'opération de suppression standard utilise `soft cancel` (statut `cancelled`) qui révoque le QR proprement.
- [ ] **AC-E3** Une suppression dure (RGPD) déclenche révocation explicite des QR et conservation de l'historique anonymisé selon politique produit.

## F. Scan et UX hôtesse

- [ ] **AC-F1** Les cas ambigus (`supervisor_review`) sont signalés visuellement et n'autorisent pas le check-in sans confirmation.
- [ ] **AC-F2** Les messages de scan sont exploitables par l'hôtesse (français clair, pas de jargon technique, pas d'UUID).
- [ ] **AC-F3** Tout scan retourne une action recommandée (`allow_check_in`, `block`, `supervisor_review`, `search_manually`).
- [ ] **AC-F4** Un ancien QR (encodant `registration.id` brut) reste accepté pendant la période de grace définie produit.
- [ ] **AC-F5** L'expérience hôtesse est testable en mode démo sans connexion à un événement réel.

## G. Tests

- [ ] **AC-G1** Tests unitaires couvrent : émission, révocation, remplacement, résolution directe, résolution fallback, refus.
- [ ] **AC-G2** Tests d'intégration couvrent : import avec match `external_reference`, match email, `needs_review`, doublon, registration absente.
- [ ] **AC-G3** Tests e2e couvrent : scan `valid`, `valid_with_warning`, `alias_resolved`, `already_checked_in`, `cancelled`, `revoked`, `event_mismatch`, `unknown_token`, `unresolved_token`, `supervisor_review`.
- [ ] **AC-G4** Tests de scénario jour J : import → envoi QR → modification → réimport → scan → alias_resolved valide.
- [ ] **AC-G5** Tests de migration : anciens QR (encodant `registration.id`) résolus correctement pendant la période de grace.

## H. Compatibilité future

- [ ] **AC-H1** La solution reste compatible avec la future architecture async BullMQ/Redis (émission asynchrone de QR via job idempotent).
- [ ] **AC-H2** La solution reste compatible avec une future architecture event-driven (publication d'événements domaine `QrIssued`, `QrRevoked`, `RegistrationCheckedIn`).
- [ ] **AC-H3** L'API de scan publie un log d'événement consommable par un futur bus.
- [ ] **AC-H4** Aucune logique critique n'est verrouillée dans Redis (la DB reste source de vérité).

## I. Sécurité & audit

- [ ] **AC-I1** Le `token` d'un `QrCredential` a une entropie ≥ 128 bits.
- [ ] **AC-I2** Le `token` ne contient aucune PII.
- [ ] **AC-I3** La route publique d'affichage QR (cas email) ne révèle plus l'UUID interne d'une registration.
- [ ] **AC-I4** Tout scan est loggé (event_id, qr_credential_id, registration_id résolue, result, user_id, timestamp).
- [ ] **AC-I5** Tout changement de statut de `QrCredential` (`active` → `revoked`, etc.) est audité avec acteur + raison.

## J. Observabilité

- [ ] **AC-J1** KPI exposés : nombre de scans par result, taux d'alias_resolved, taux de needs_review, latence p95 du scan.
- [ ] **AC-J2** Dashboard produit minimal : émission, envoi, scans, refus, alias.
- [ ] **AC-J3** Alertes sur taux anormal de `unresolved_token` ou `supervisor_review`.

---

> Ces critères sont une **base de travail**. Ils seront raffinés après validation produit des questions ouvertes ([09-open-questions.md](./09-open-questions.md)).
