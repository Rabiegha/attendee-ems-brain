# Workstream — Onboarding & Billing Management

```
Initiative: SaaS Foundation
Workstream: Onboarding & Billing Management
Status:     Backlog / Discovery
Priority:   Medium
```

## 1. Objectif

Construire les fondations SaaS du produit Attendee :
- Onboarding multi-tenant (création d'organization, invitations, gestion membership).
- Visibilité plan / subscription côté tenant.
- Console Platform Admin (gestion plateforme côté éditeur).
- Public namespaces (slugs publics séparés du slug interne org).

## 2. Status

**Backlog / Discovery.**

> ⚠️ Ce workstream **n'est pas le focus actuel**. Le focus actuel reste **Async Architecture** (voir [docs/steering/NOW.md](../../steering/NOW.md)).

Aucune implémentation n'est lancée. Le travail consiste à :
- Cadrer le scope.
- Documenter les décisions déjà prises ([../../steering/DECISIONS.md](../../steering/DECISIONS.md)).
- Préparer les epics pour quand le moment sera venu.

## 3. Backlog items rattachés

| Backlog item | Fiche détaillée |
|---|---|
| Onboarding multi-tenant | [../../backlog/onboarding-multi-tenant.md](../../backlog/onboarding-multi-tenant.md) |
| Billing & plan management | [../../backlog/billing-plan-management.md](../../backlog/billing-plan-management.md) |
| Platform Admin Console | [../../backlog/platform-admin-console.md](../../backlog/platform-admin-console.md) |
| Public Landing Page Namespaces | [../../backlog/public-landing-page-namespaces.md](../../backlog/public-landing-page-namespaces.md) |

## 4. Documents référents existants

### Architecture
- [docs/architecture/organization-plan-subscription-model.md](../../architecture/organization-plan-subscription-model.md)
- [docs/architecture/platform-admin-access-model.md](../../architecture/platform-admin-access-model.md)

### Product
- [docs/product/saas-foundation-onboarding-billing.md](../../product/saas-foundation-onboarding-billing.md)

### Roadmap
- [docs/roadmap/saas-foundation-epics.md](../../roadmap/saas-foundation-epics.md)

## 5. Décisions déjà actées

Voir [docs/steering/DECISIONS.md](../../steering/DECISIONS.md) :

- User / Organization / Membership model.
- Plan ownership (par Organization, pas par User).
- Free plan rule (une seule Org Free créée par User).
- Platform Admin séparé des rôles tenant.
- Public namespaces séparés du `internalSlug`.

## 6. Rabbit holes à éviter

- [full-stripe-billing.md](../../rabbit-holes/full-stripe-billing.md)
- [separate-admin-frontend.md](../../rabbit-holes/separate-admin-frontend.md)
- [full-custom-domains.md](../../rabbit-holes/full-custom-domains.md)
- [user-account-deletion-anonymization.md](../../rabbit-holes/user-account-deletion-anonymization.md)

## 7. Epics envisagés (non encore lancés)

| Epic | Couvre |
|---|---|
| Organization onboarding | Création org, premier utilisateur, choix plan, invitations. |
| Plan and subscription visibility | UI compteurs limites, page billing read-only. |
| Platform admin foundation | Section `/platform` dans frontend existant. |
| Public namespace foundation | Modèle `PublicNamespace`, réservation slugs, validation. |

## 8. Hors scope (de ce workstream et pour l'instant)

- Intégration Stripe complète (paiement, webhooks, dunning).
- Frontend admin séparé (deuxième app).
- Custom domains complets (DNS, TLS automation).
- Impersonation Platform Admin → tenant.
- Anonymisation / suppression complète des données User à l'échelle plateforme.

## 9. Quand reprendre

Quand le workstream **Async Architecture** aura atteint un palier stable (Q&A clos + Palier 1 conventions actées). Pas avant.

## 10. Notes pour Copilot

- Ce workstream est **discovery uniquement**. Aucune implémentation Stripe, Platform Admin, Public Namespaces sans ticket explicite.
- Tout sujet précis qui émerge doit être ajouté en fiche dans [../../backlog/](../../backlog/).
- Tout sujet identifié comme tentant mais risqué doit aller dans [../../rabbit-holes/](../../rabbit-holes/).
