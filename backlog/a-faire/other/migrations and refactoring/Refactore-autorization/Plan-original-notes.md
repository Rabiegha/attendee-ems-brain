# Plan de refactoring — Authorization

> Stratégie en 3 phases. On ne passe à la phase suivante que quand la précédente est stable en prod.

---

## Phase 1 — Tenant-first + root override (PRIORITAIRE)

**Objectif** : un système d'autorisation fonctionnel pour 100% des cas métier actuels.

### Ce qu'on fait

| Sujet | Action |
|-------|--------|
| **Rôles tenant** | Seul système de rôles actif. 1 rôle par user par org (ORG_ADMIN, ORG_MANAGER, ORG_STAFF, ORG_VIEWER). |
| **Root système** | Sorti du modèle `Role`. Stocké dans `UserSystemAccess` (table dédiée, auditée). Bypass toutes les permissions, mais DOIT avoir un `activeOrgId`. |
| **Multi-org** | Un user peut appartenir à N orgs via `OrgUser`. Au login multi-org → token temporaire sans org → choix org → `switch-org` → nouveau JWT avec context. |
| **Switch-org** | Endpoint `POST /auth/switch-org`. Vérifie l'accès (membership ou root), génère un nouveau JWT avec `currentOrgId` + permissions du rôle dans cette org. |
| **event_access** | Table de liaison user ↔ event pour le scope `assigned`. Utilisée par le `ScopeEvaluator`. |
| **Platform gelée** | `PlatformUserRole` et `PlatformUserOrgAccess` restent en base mais sont ignorées par le runtime. Plus de `mode: platform` dans les JWT. |
| **Audit** | Logs d'audit minimaux : login, switch-org, actions root. |

### Ce qu'on ne fait PAS

- Pas de rôles internes (support, ops)
- Pas de gestion platform dans le runtime
- Pas de rôles personnalisés par org

### Livrables

- [ ] `UserSystemAccess` en base + migration
- [ ] JWT unifié (`mode` supprimé, toujours tenant-context)
- [ ] Endpoints `GET /auth/me/orgs` + `POST /auth/switch-org`
- [ ] `TenantContextGuard` activé sur toutes les routes tenant
- [ ] `AuthorizationService.can()` vérifie root via `UserSystemAccess`
- [ ] Audit service minimal

---

## Phase 2 — Comptes internes non-root

**Objectif** : ajouter une couche "staff interne" sans polluer les rôles tenant.

### Ce qu'on ajoute

- Rôles internes dédiés : `SUPPORT`, `OPS`, éventuellement `SALES`, `ONBOARDING`
- Ces rôles ne sont PAS des rôles tenant — ils ne donnent pas accès aux fonctions métier comme un ORG_ADMIN
- Permissions spécifiques au support : lecture cross-org, impersonate, debug
- Accès aux orgs via assignation explicite (pas d'accès global sauf root)

### Principes

- Un staff interne qui travaille dans une org le fait avec un rôle interne, jamais un rôle tenant
- Séparation stricte : les rôles internes ne peuvent pas créer/modifier les données métier (attendees, registrations) sauf si explicitement autorisé
- Toujours scopé à une org active (pas de mode "platform global")

---

## Phase 3 — Accès platform structurés

**Objectif** : un modèle platform explicite, auditable, avec gestion fine des accès.

### Ce qu'on ajoute

- Modèle d'accès platform formel :
  - **Global** : accès à toutes les orgs (root uniquement)
  - **Limité** : accès à certaines orgs assignées
  - **Temporaire** : accès avec expiration automatique
- Audit complet de toutes les actions platform
- Dashboard d'administration des accès platform
- Mécanisme d'expiration et de révision périodique des accès