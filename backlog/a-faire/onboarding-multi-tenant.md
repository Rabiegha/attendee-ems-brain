# Backlog — Onboarding multi-tenant

Type: Backlog Item
Initiative: SaaS Foundation
Workstream: Onboarding & Billing Management
Status: Backlog
Priority: Medium

## 1. Why this exists

Attendee est un produit SaaS multi-tenant. Un utilisateur peut créer plusieurs organisations, être invité à des organisations existantes, et basculer entre elles.

Aujourd'hui, des briques existent (User, Organization, OrgUser, invitations) mais le **parcours complet d'onboarding** (création org → choix plan → premier membre → invitations → bascule org active) n'est pas formalisé comme une expérience cohérente.

## 2. Current understanding

- Modèle DB : `User`, `Organization`, `OrgUser` (composite `[user_id, org_id]`), `Invitation` (avec `InvitationStatus` PENDING/ACCEPTED/EXPIRED/CANCELLED).
- JWT porte `userId` + l'`orgId` actif.
- Front gère la bascule d'org via re-login ou refresh du contexte (à confirmer côté front).
- Une règle métier validée existe : un User ne peut avoir qu'**une seule Organization Free active créée par lui** (voir [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#free-plan-rule)).

## 3. What is already decided

Voir [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md) :
- User = identité globale, Organization = espace de travail, Membership = lien.
- Données métier appartiennent à Organization.
- Free plan rule.
- Platform Admin est séparé des rôles tenant.

## 4. What is not decided yet

- Parcours UX exact d'inscription (auto-création d'org au signup ? choix plan immédiat ?).
- Page « mes organisations » côté front (existe-t-elle ? scope ?).
- Stratégie d'invitation : email + lien magique ? expiration ? réutilisation ?
- Comportement quand un User est désinvité / quitte une org (lien avec [docs/bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md)).
- Limites par plan applicables dès l'onboarding (compteur events, registrations, prints, users).

## 5. Why not now

Le focus actuel est **Async Architecture** (voir [../steering/NOW.md](../../../attendee-ems-back/docs/steering/NOW.md)). Toucher au parcours onboarding sans avoir stabilisé les fondations async + bugs sécurité ouverts (CORS, user deletion) introduirait du risque inutile.

De plus, beaucoup de décisions d'onboarding dépendent de Billing (cf. [billing-plan-management.md](billing-plan-management.md)) — il vaut mieux les traiter ensemble.

## 6. When to revisit

- Après clôture de la phase Q&A Async Architecture.
- Après décision humaine sur les questions BLOCKING listées dans [../async-architecture/QandA/12-open-questions-for-human.md](../../workstreams/en-cours/async-architecture/QandA/12-open-questions-for-human.md).
- Avant l'epic « Plan and subscription visibility » côté Billing (les deux se croisent).

## 7. Related docs

- [../architecture/organization-plan-subscription-model.md](../../architecture/organization-plan-subscription-model.md)
- [../architecture/platform-admin-access-model.md](../../architecture/platform-admin-access-model.md)
- [../product/saas-foundation-onboarding-billing.md](../../product/saas-foundation-onboarding-billing.md)
- [../roadmap/saas-foundation-epics.md](saas-foundation-epics.md)
- [../workstreams/onboarding-billing-management/README.md](../../workstreams/en-cours/async-architecture/README.md)
- [../bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md)

## 8. Possible future epics

- **Organization onboarding** : signup → première org → choix plan → invitations initiales.
- **Membership management** : invitations, rôles, départ, transfert ownership.
- **Org switcher** : bascule fluide entre organisations côté front.
- **Free plan enforcement** : validation backend de la règle « une seule org Free créée par user ».

## 9. Notes for Copilot

- Ne pas créer de migration Prisma sur User / Organization / OrgUser / Invitation depuis ce backlog.
- Ne pas câbler de nouveau parcours d'onboarding sans ticket explicite.
- Si une question d'onboarding émerge pendant un autre chantier, l'**ajouter ici** au lieu de coder.
- Lier les décisions actées dans [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md).
