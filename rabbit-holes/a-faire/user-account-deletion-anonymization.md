# Rabbit Hole — User account deletion / anonymization now

Type: Rabbit Hole
Status: Watchlist
Risk level: High

## 1. Why it is tempting

- Conformité RGPD (« droit à l'oubli »).
- Demande probable d'un futur client enterprise.
- Le bug review [../bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md) attire déjà l'attention sur le sujet.
- Tentation de « faire vite » une suppression cascade pour clôturer le bug.

## 2. Why it is dangerous now

- Un User est lié à **plusieurs Organizations** (via `OrgUser`) → supprimer un User a des effets cross-tenant.
- Les données métier (registrations, badges, prints, exports, audit) référencent des `user_id` partout — supprimer = casser l'historique des autres orgs.
- Aucune politique **claire** sur quoi anonymiser, quoi supprimer, quoi conserver pour audit légal.
- Aucun **AuditLog** en place → impossible de prouver post-mortem ce qui a été supprimé.
- Aucun **EmailLog** → confirmation d'envoi des emails de suppression non traçable.
- Le sujet exige une **décision produit + légale** (RGPD, conservation comptable), pas juste technique.
- Une mauvaise suppression cascade = perte irréversible de données prod.

## 3. Scope creep risk

Très élevé :
- Politique de conservation par type d'entité.
- Anonymisation pseudo-réversible vs suppression définitive.
- Re-création du compte avec le même email (collision).
- Export complet des données utilisateur (RGPD article 20).
- UI tenant + UI Platform Admin pour gérer la demande.
- Webhook vers intégrations externes (n8n).
- Tests de non-régression sur tous les modules qui référencent `user_id`.

## 4. Current decision

- **Pas de suppression / anonymisation globale maintenant.**
- Le **bug review** [../bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md) reste **une analyse**, pas un ordre de correction.
- Aucune action de cascade ou de soft-delete cross-tenant à ajouter sans ticket explicite et décision produit/légale.

## 5. When to revisit

- Quand un client (idéalement payant) déclenche une demande RGPD formelle.
- Après création du modèle `AuditLog` (pour tracer toute suppression).
- En tandem avec une décision produit / légale claire (politique de conservation).

## 6. Related docs

- [../bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md)
- [../backlog/onboarding-multi-tenant.md](../../backlog/a-faire/onboarding-multi-tenant.md)
- [../async-architecture/QandA/13-information-gaps.md](../../workstreams/a-faire/async-architecture/QandA/13-information-gaps.md)
- [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#bug-handling-rule)

## 7. Rule for Copilot

- Si une proposition contient « supprimer un User », « cascade delete », « anonymiser un compte », « RGPD delete » → **stop**, pointer vers ce document.
- Interdit : modifier la relation `User ↔ OrgUser` ou ajouter un endpoint `DELETE /users/me` sans ticket et décision validée.
- Autorisé : compléter le bug review avec les impacts identifiés.
- La présence du bug review **n'est pas** un mandat pour corriger.
