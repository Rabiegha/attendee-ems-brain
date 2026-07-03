# Rabbit Hole — Full Stripe billing now

Type: Rabbit Hole
Status: Watchlist
Risk level: High

## 1. Why it is tempting

- Le projet est un SaaS → forcément il faudra Stripe un jour.
- Stripe est bien documenté, il existe des libs NestJS prêtes.
- On peut être tenté de « tout brancher » d'un coup (checkout, subscriptions, invoices, webhooks, dunning, tax).
- Une décision de plan ownership (Organization, pas User) est déjà actée → on peut penser que le terrain est prêt.

## 2. Why it is dangerous now

- Aucun **plan** formellement défini (prix, limites, périodicité, trial).
- Aucun **enforcement** des limites côté backend.
- Aucun **AuditLog** pour tracer les changements de plan / facturation.
- Aucun **EmailLog** pour prouver l'envoi des emails Stripe-déclenchés.
- L'**onboarding multi-tenant** n'est pas cadré ([../backlog/onboarding-multi-tenant.md](../backlog/onboarding-multi-tenant.md)) → où poser l'UI billing ?
- Stripe **complet** = checkout + portal + webhooks + retry + dunning + tax + invoices + refunds + plan changes + proration. Énorme surface.
- Une intégration mal câblée = vraie perte d'argent / litige client.
- Aucun besoin commercial urgent identifié.

## 3. Scope creep risk

Très élevé :
- Webhooks Stripe → besoin d'une queue dédiée et d'idempotence forte.
- Plan changes → migrations DB, gestion des proratas.
- Trial → flow d'expiration + downgrade + bloc.
- Tax → règles européennes, VAT, factures conformes.
- Tableau de bord billing tenant + côté Platform Admin.
- Connexion aux limites enforced → touche tous les services qui créent des entités.

## 4. Current decision

- **Pas maintenant.**
- Le **backlog billing** ([../backlog/billing-plan-management.md](../backlog/billing-plan-management.md)) ne contient **pas** d'épic Stripe complet.
- Si un premier client payant arrive, démarrer par une **intégration Stripe minimale** (checkout + 1 webhook + statut de subscription read-only) — surtout pas l'intégration full.

## 5. When to revisit

- Quand un **premier client payant concret** se présente.
- Après stabilisation de l'onboarding multi-tenant.
- Après création des modèles `Plan` / `Subscription` / `AuditLog` / `EmailLog`.

## 6. Related docs

- [../backlog/billing-plan-management.md](../backlog/billing-plan-management.md)
- [../architecture/organization-plan-subscription-model.md](../architecture/organization-plan-subscription-model.md)
- [../product/saas-foundation-onboarding-billing.md](../product/saas-foundation-onboarding-billing.md)
- [../roadmap/saas-foundation-epics.md](../roadmap/saas-foundation-epics.md)
- [../steering/DECISIONS.md](../steering/DECISIONS.md#plan-ownership)

## 7. Rule for Copilot

- Si une proposition contient « intégrer Stripe », « ajouter checkout », « ajouter webhooks billing » → **stop**, pointer vers ce document.
- Autorisé : documenter dans [../backlog/billing-plan-management.md](../backlog/billing-plan-management.md) ce qu'il faudra prévoir.
- Interdit : créer un module `BillingModule`, ajouter des routes `/stripe/*`, installer `stripe` en dépendance, créer des migrations Prisma sur `Subscription`, sans ticket explicite et sans décision actée sur les plans.
