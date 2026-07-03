# Backlog — Billing & plan management

Type: Backlog Item
Initiative: SaaS Foundation
Workstream: Onboarding & Billing Management
Status: Backlog
Priority: Medium

## 1. Why this exists

Attendee a vocation à être un SaaS payant avec plusieurs plans (Free, Trial, payants). Il faut :
- Afficher au tenant son plan actif et ses compteurs de limites.
- Empêcher (ou avertir) les dépassements de quota.
- Préparer (sans implémenter complètement) l'intégration Stripe pour gérer abonnements, paiements, factures.

## 2. Current understanding

- Le modèle `Plan` / `Subscription` (ou équivalent) doit appartenir à `Organization`, **pas à User** (décision actée, voir [../steering/DECISIONS.md](../steering/DECISIONS.md#plan-ownership)).
- Aucune intégration paiement n'est en place aujourd'hui.
- Les limites (events, registrations, prints, etc.) ne sont pas formalisées de façon centralisée.
- Une règle Free plan existe (voir DECISIONS).

## 3. What is already decided

- **Plan ownership = Organization** (pas User).
- Vérification des limites sur `currentOrgId`.
- Free plan rule : une seule org Free active créée par User.
- Stripe complet **maintenant** est un rabbit hole — voir [../rabbit-holes/full-stripe-billing.md](../rabbit-holes/full-stripe-billing.md).

## 4. What is not decided yet

- Liste exacte des plans (noms, prix, limites).
- Périmètre des limites enforced backend vs juste affichées.
- Intégration paiement : Stripe ? Lemon Squeezy ? Autre ?
- Gestion des invoices, dunning, downgrades automatiques.
- Période de trial : durée, conversion auto, blocage.
- Comportement à l'expiration / au dépassement de plan.

## 5. Why not now

- Focus actuel = Async Architecture (voir [../steering/NOW.md](../steering/NOW.md)).
- Implémentation Stripe complète est un rabbit hole (lourd, à part).
- Sans onboarding multi-tenant cadré, billing n'a pas de surface UI cohérente où atterrir.

## 6. When to revisit

- Après stabilisation du workstream Async Architecture.
- En tandem avec [onboarding-multi-tenant.md](onboarding-multi-tenant.md).
- Quand un besoin commercial (premier client payant) le rend incontournable.

## 7. Related docs

- [../architecture/organization-plan-subscription-model.md](../architecture/organization-plan-subscription-model.md)
- [../product/saas-foundation-onboarding-billing.md](../product/saas-foundation-onboarding-billing.md)
- [../roadmap/saas-foundation-epics.md](../roadmap/saas-foundation-epics.md)
- [../workstreams/onboarding-billing-management/README.md](../workstreams/async-architecture/README.md)
- [../rabbit-holes/full-stripe-billing.md](../rabbit-holes/full-stripe-billing.md)

## 8. Possible future epics

- **Plan and subscription visibility** : page tenant lecture seule (plan actuel, compteurs, prochain renouvellement).
- **Limit enforcement** : middleware / guard de vérification des limites par `currentOrgId`.
- **Stripe minimal integration** : création checkout session + webhook minimal (séparé d'une vraie facturation complète).
- **Trial flow** : trial automatique à la création, conversion ou downgrade à l'expiration.

## 9. Notes for Copilot

- **Ne pas câbler Stripe** depuis ce backlog. Toute proposition Stripe doit pointer vers [../rabbit-holes/full-stripe-billing.md](../rabbit-holes/full-stripe-billing.md).
- **Ne pas créer** de modèle Prisma `Plan` / `Subscription` sans ticket et sans décision sur les plans.
- Si une vérif de limite est ajoutée ailleurs (ex : import bulk), la documenter ici pour rester cohérent.
