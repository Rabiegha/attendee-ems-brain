# Backlog — Platform Admin Console

Type: Backlog Item
Initiative: SaaS Foundation
Workstream: Onboarding & Billing Management
Status: Backlog
Priority: Medium

## 1. Why this exists

L'équipe éditeur a besoin d'une console pour :
- Voir la liste des Organizations.
- Voir les utilisateurs, plans, usages.
- Intervenir en support (sans impersonation magique).
- Surveiller la santé plateforme (Bull Board, ExportJobs, etc.).

Aujourd'hui : pas de console dédiée. L'accès se fait via Bull Board, requêtes DB directes, ou outils ad hoc.

## 2. Current understanding

- Décision actée : **Platform Admin est séparé des rôles tenant** (voir [../steering/DECISIONS.md](../steering/DECISIONS.md#platform-admin)).
- Un Platform Admin n'est **pas** Owner magique de toutes les Orgs.
- Pour le **MVP**, préférer une section `/platform` **dans le frontend existant** plutôt qu'un nouveau frontend séparé (voir rabbit hole [../rabbit-holes/separate-admin-frontend.md](../rabbit-holes/separate-admin-frontend.md)).
- Documents architecture existants : [../architecture/platform-admin-access-model.md](../architecture/platform-admin-access-model.md).

## 3. What is already decided

- Séparation Platform Admin ↔ rôles tenant.
- Pas d'impersonation pour le MVP.
- Pas de second frontend pour le MVP.
- Bull Board doit être protégé (cf. [../async-architecture/QandA/09-security-multitenancy.md](../workstreams/async-architecture/QandA/09-security-multitenancy.md)) — préférer Guard SUPER_ADMIN + basic-auth Nginx.

## 4. What is not decided yet

- Périmètre exact de la console MVP (lecture seule ? actions limitées ?).
- Mécanisme d'auth Platform Admin (claim JWT dédié ? table `PlatformAdmin` ?).
- Audit des actions Platform Admin (besoin d'un `AuditLog`, cf. [../async-architecture/QandA/13-information-gaps.md](../workstreams/async-architecture/QandA/13-information-gaps.md)).
- Intégration de Bull Board dans la console ou maintien séparé.

## 5. Why not now

- Focus actuel = Async Architecture.
- Sans `AuditLog`, toute action Platform Admin serait non traçable — bug en puissance.
- Sans onboarding cadré, certaines vues admin n'auraient pas de sens.

## 6. When to revisit

- Après décision sur `AuditLog` (Q11 du QandA Async Architecture).
- Quand un besoin support concret apparaît (premier client payant, premier ticket de support qui demande l'accès admin).
- En tandem avec [billing-plan-management.md](billing-plan-management.md) pour piloter les plans côté éditeur.

## 7. Related docs

- [../architecture/platform-admin-access-model.md](../architecture/platform-admin-access-model.md)
- [../product/saas-foundation-onboarding-billing.md](../product/saas-foundation-onboarding-billing.md)
- [../roadmap/saas-foundation-epics.md](../roadmap/saas-foundation-epics.md)
- [../workstreams/onboarding-billing-management/README.md](../workstreams/async-architecture/README.md)
- [../rabbit-holes/separate-admin-frontend.md](../rabbit-holes/separate-admin-frontend.md)

## 8. Possible future epics

- **Platform admin foundation** : table `PlatformAdmin` ou claim JWT, Guard `SuperAdminGuard`, section `/platform` MVP.
- **Org overview** : liste paginée des orgs avec plan, usage, owner.
- **Job dashboard intégré** : Bull Board derrière le Guard SUPER_ADMIN.
- **Audit log viewer** (dépend de la création du modèle `AuditLog`).

## 9. Notes for Copilot

- **Ne pas créer** de seconde app frontend.
- **Ne pas câbler** d'impersonation.
- Toute proposition de Platform Admin doit s'appuyer sur [../architecture/platform-admin-access-model.md](../architecture/platform-admin-access-model.md).
- Toute action admin devra **logger un AuditLog** — mais le modèle n'existe pas encore, donc rien à câbler maintenant.
