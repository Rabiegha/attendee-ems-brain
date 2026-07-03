# Backlog — Public Landing Page Namespaces

Type: Backlog Item
Initiative: SaaS Foundation
Workstream: Onboarding & Billing Management
Status: Backlog
Priority: Medium

## 1. Why this exists

À terme, chaque Organization doit pouvoir avoir une **page d'atterrissage publique** (formulaire d'inscription public, page event publique) accessible via un **slug public** propre :
- `acme.attendee.app/event/xxx`
- ou `acme.attendee.app`

Ce slug public ne doit **pas être confondu** avec l'identifiant interne d'organisation (utilisé pour le scoping data).

## 2. Current understanding

- Aucun mécanisme `PublicNamespace` distinct n'existe aujourd'hui — l'URL publique de Public-Form-Logger utilise un identifiant d'event ou un token.
- Le repo `Public-Form-Logger/` héberge déjà la couche publique de saisie formulaire (séparée de l'API back métier).
- Décisions déjà actées (voir [../steering/DECISIONS.md](../steering/DECISIONS.md#public-namespaces)) :
  - Ne **pas** utiliser `Organization.slug` directement comme sous-domaine garanti.
  - Séparer `Organization.internalSlug` et `PublicNamespace.subdomain`.
  - Les slugs **premium / sensibles** doivent être réservés / validés.

## 3. What is already decided

- Séparation `internalSlug` (interne, immutable, technique) et `subdomain` public.
- Slugs premium et sensibles réservés.
- Pas de custom domain complet pour l'instant (rabbit hole — voir [../rabbit-holes/full-custom-domains.md](../rabbit-holes/full-custom-domains.md)).

## 4. What is not decided yet

- Schéma exact du modèle `PublicNamespace` (id, subdomain, org_id, status, reserved_until…).
- Liste précise des slugs réservés (admin, www, api, app, support, billing, etc.).
- Politique de changement de subdomain (autorisé ? avec redirect ?).
- Cas multi-namespaces par org (sous-marques, événements white-label).
- Workflow de validation (auto-approve, modération manuelle pour certains slugs ?).
- Lien avec le routage Public-Form-Logger.

## 5. Why not now

- Focus actuel = Async Architecture.
- Pas de demande client urgente.
- Custom domains complets sont un rabbit hole — autant ne pas démarrer un demi-chantier.

## 6. When to revisit

- Quand un client demande une URL personnalisée.
- En tandem avec [onboarding-multi-tenant.md](onboarding-multi-tenant.md) (choix de slug à la création d'org).
- Avant toute communication marketing qui promettrait une URL custom.

## 7. Related docs

- [../architecture/organization-plan-subscription-model.md](../architecture/organization-plan-subscription-model.md)
- [../product/saas-foundation-onboarding-billing.md](../product/saas-foundation-onboarding-billing.md)
- [../roadmap/saas-foundation-epics.md](../roadmap/saas-foundation-epics.md)
- [../rabbit-holes/full-custom-domains.md](../rabbit-holes/full-custom-domains.md)
- Repo `Public-Form-Logger/` (architecture publique existante).

## 8. Possible future epics

- **Public namespace foundation** : modèle `PublicNamespace`, table de slugs réservés, validation.
- **Subdomain routing** : middleware/edge qui résout `acme.attendee.app` → `org_id`.
- **Slug change workflow** : gestion redirect / collision.
- **Org public landing page** : page d'index publique par namespace.

## 9. Notes for Copilot

- **Ne pas** ajouter de champ `subdomain` à `Organization` ; passer par un modèle dédié.
- **Ne pas** utiliser `Organization.slug` comme route publique sans validation préalable.
- Toute proposition « custom domain » doit aller dans [../rabbit-holes/full-custom-domains.md](../rabbit-holes/full-custom-domains.md).
- Ne pas créer de migration Prisma depuis ce backlog.
