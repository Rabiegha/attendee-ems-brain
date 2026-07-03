# Diagnostic Architecture Hexagonale — État des lieux complet

> Date : 24 février 2026  
> Scope : Backend NestJS (`attendee-ems-back/src/`)  
> Objectif : Documenter l'état de la migration vers l'architecture hexagonale, identifier les risques, et fournir un plan de migration complet.

---

## Table des matières

1. [État actuel de l'architecture](#1-état-actuel-de-larchitecture)
2. [Les deux systèmes d'autorisation](#2-les-deux-systèmes-dautorisation)
3. [Cartographie complète module par module](#3-cartographie-complète-module-par-module)
4. [Code mort identifié](#4-code-mort-identifié)
5. [Risques de l'architecture actuelle](#5-risques-de-larchitecture-actuelle)
6. [Bénéfices du passage en full hexagonal](#6-bénéfices-du-passage-en-full-hexagonal)
7. [Plan de migration](#7-plan-de-migration)

---

## 1. État actuel de l'architecture

### 1.1 Structure du dossier `src/`

```
src/
├── app.module.ts
├── main.ts
├── auth/                    ← JWT login/register (AuthModule)
├── authorization/           ← ⚠️ ANCIEN système CASL — CODE MORT
│   ├── casl.module.ts       ← @Global, fournit CaslAbilityFactory
│   ├── casl-ability.factory.ts
│   ├── rbac.module.ts       ← Doublon de casl.module sans @Global()
│   └── rbac.service.ts      ← Jamais utilisé nulle part
├── platform/
│   └── authz/               ← ✅ NOUVEAU système hexagonal — LE VRAI
│       ├── core/
│       ├── ports/
│       ├── adapters/
│       └── authz.module.ts
├── common/
│   ├── guards/
│   │   ├── jwt-auth.guard.ts
│   │   └── permissions.guard.ts  ← Ancien décorateur @Permissions(), NOUVEAU moteur
│   ├── decorators/
│   │   └── permissions.decorator.ts  ← @Permissions() (ancien décorateur)
│   └── utils/
│       ├── org-scope.util.ts               ← Actif
│       ├── resolve-attendee-scope.util.ts  ← Actif, référence :org (problème)
│       ├── resolve-event-scope.util.ts     ← Actif, référence :org (problème)
│       ├── resolve-registration-scope.util.ts ← Actif
│       └── resolve-user-scope.util.ts      ← ❌ CODE MORT — jamais importé
├── modules/                 ← Modules métier (architecture plate NestJS standard)
│   ├── attendees/
│   ├── attendee-types/
│   ├── badge-generation/
│   ├── badge-templates/
│   ├── badges/
│   ├── email/
│   ├── events/
│   ├── invitations/
│   ├── organizations/
│   ├── partner-scans/       ← Nouveau module (24 fév 2026)
│   ├── permissions/
│   ├── public/
│   ├── registrations/
│   ├── roles/
│   ├── sessions/
│   ├── tags/
│   └── users/
├── infra/                   ← Prisma, Storage
├── config/
├── health/
├── print-queue/
├── router/
└── websocket/
```

### 1.2 Constat principal

**La migration vers `platform/authz` est pratiquement terminée au niveau du moteur d'autorisation**. Tous les contrôleurs (15 sur 15) utilisent le nouveau moteur RBAC. Cependant, il reste du **code mort** et des **imports résidus** de l'ancien système qui créent de la confusion.

---

## 2. Les deux systèmes d'autorisation

### 2.1 ANCIEN — `src/authorization/` (CASL)

| Fichier | Ce qu'il fait | Utilisé ? |
|---------|--------------|-----------|
| `casl.module.ts` | Module `@Global()`, fournit `CaslAbilityFactory` | Importé par 4 modules, mais **CaslAbilityFactory n'est injectée nulle part** |
| `casl-ability.factory.ts` | Convertit permissions en abilities CASL (mais **ignore les scopes**) | ❌ **Jamais injecté dans aucun controller/service** |
| `rbac.module.ts` | Doublon de CaslModule sans `@Global()` | Importé par 4 modules, mêmes conclusions |
| `rbac.service.ts` | Moteur RBAC alternatif avec résolution de scope | ❌ **Jamais importé nulle part** — complètement mort |

#### Modules qui importent encore l'ancien système (résidus)

| Module | Import résidu | Impact réel |
|--------|-------------|-------------|
| `registrations.module.ts` | `CaslModule` | ❌ Aucun — `CaslAbilityFactory` pas utilisée |
| `email.module.ts` | `CaslModule` | ❌ Aucun |
| `events.module.ts` | `CaslModule` | ❌ Aucun |
| `partner-scans.module.ts` | `CaslModule` | ❌ Aucun |
| `attendees.module.ts` | `RbacModule` | ❌ Aucun |
| `organizations.module.ts` | `RbacModule` | ❌ Aucun |
| `roles.module.ts` | `RbacModule` | ❌ Aucun |
| `permissions.module.ts` | `RbacModule` | ❌ Aucun |

**Verdict : L'ancien système est 100% mort mais 8 modules traînent encore ses imports.**

### 2.2 NOUVEAU — `src/platform/authz/` (Hexagonal)

```
platform/authz/
├── authz.module.ts                          ← @Global(), câble les ports aux adapters
│
├── core/                                    ← ZÉRO dépendance NestJS/Prisma
│   ├── types.ts                             ← AuthContext, RbacContext, Grant, Scope
│   ├── authorization.service.ts             ← Moteur : can(), canAll(), canAny()
│   ├── decision.ts                          ← Enum DecisionCode + helpers
│   ├── permission-resolver.ts               ← Résout grants depuis userId/orgId
│   └── scope-evaluator.ts                   ← Évalue any/assigned/own
│
├── ports/                                   ← Interfaces pures (contrats)
│   ├── rbac-query.port.ts                   ← getTenantRole, getGrants...
│   ├── membership.port.ts                   ← isMemberOfOrg
│   ├── module-gating.port.ts                ← isModuleEnabled (MVP: always true)
│   └── auth-context.port.ts                 ← buildAuthContext(jwtPayload)
│
├── adapters/
│   ├── db/                                  ← Implémentations Prisma des ports
│   │   ├── prisma-rbac-query.adapter.ts
│   │   ├── prisma-membership.adapter.ts
│   │   ├── prisma-module-gating.adapter.ts
│   │   └── prisma-auth-context.adapter.ts
│   └── http/
│       ├── controllers/
│       │   ├── me-ability.controller.ts     ← GET /me/ability
│       │   └── rbac-admin.controller.ts     ← Admin RBAC endpoints
│       ├── guards/
│       │   └── require-permission.guard.ts  ← LE guard principal
│       └── decorators/
│           └── require-permission.decorator.ts ← @RequirePermission()
│
└── permission-registry.ts
```

#### Câblage dans `authz.module.ts`

```typescript
// Injection de dépendance hexagonale via NestJS DI
{ provide: RBAC_QUERY_PORT,   useClass: PrismaRbacQueryAdapter }
{ provide: MEMBERSHIP_PORT,   useClass: PrismaMembershipAdapter }
{ provide: MODULE_GATING_PORT, useClass: PrismaModuleGatingAdapter }
{ provide: AUTH_CONTEXT_PORT,  useClass: PrismaAuthContextAdapter }

// Core services construits via factory
AuthorizationService  → useFactory(permissionResolver, membershipPort)
PermissionResolver    → useFactory(rbacQueryPort)
```

**C'est exactement le pattern hexagonal** :
- `core/` ne dépend de RIEN (ni NestJS, ni Prisma)
- `ports/` définissent les contrats (interfaces TypeScript + Symbols)
- `adapters/` implémentent les ports avec Prisma/HTTP
- `authz.module.ts` câble tout via NestJS DI (`provide/useClass`)

### 2.3 Deux décorateurs coexistent

| Décorateur | Fichier | Guard utilisé | Moteur RBAC | Nb contrôleurs |
|------------|---------|-------------|-------------|----------------|
| `@RequirePermission()` | `platform/authz/adapters/http/decorators/` | `RequirePermissionGuard` | `AuthorizationService` (nouveau) | **13** |
| `@Permissions()` | `common/decorators/permissions.decorator.ts` | `PermissionsGuard` | `AuthorizationService` (nouveau aussi !) | **2** |

**Point important** : Même l'ancien décorateur `@Permissions()` utilise le **nouveau** moteur `AuthorizationService`. La différence est uniquement syntaxique :
- `@RequirePermission('events.read')` → une permission, AND
- `@Permissions('badges.create:any', 'badges.create:org')` → plusieurs permissions, OR

Les 2 contrôleurs encore sur `@Permissions()` sont : `attendee-types`, `badge-generation`.

---

## 3. Cartographie complète module par module

| Module | Import ancien | Guard | Décorateur | Système réel | Action requise |
|--------|-------------|-------|------------|-------------|----------------|
| **attendees** | `RbacModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `RbacModule` |
| **attendee-types** | — | `PermissionsGuard` | `@Permissions` | ✅ Nouveau (via pont) | Migrer vers `@RequirePermission` |
| **badge-generation** | — | `PermissionsGuard` | `@Permissions` | ✅ Nouveau (via pont) | Migrer vers `@RequirePermission` |
| **badge-templates** | — | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Rien à faire |
| **badges** | — | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Rien à faire |
| **email** | `CaslModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `CaslModule` |
| **events** | `CaslModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `CaslModule` |
| **invitations** | — | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Rien à faire |
| **organizations** | `RbacModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `RbacModule` |
| **partner-scans** | `CaslModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `CaslModule` |
| **permissions** | `RbacModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `RbacModule` |
| **public** | — | Aucun | — | N/A (routes publiques) | Rien à faire |
| **registrations** | `CaslModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `CaslModule` |
| **roles** | `RbacModule` ⚠️ | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Supprimer import `RbacModule` |
| **sessions** | — | `JwtAuthGuard` seul | — | ⚠️ **Aucun authz** | Décider : ajouter permissions ou laisser JWT-only |
| **tags** | — | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Rien à faire |
| **users** | — | `RequirePermissionGuard` | `@RequirePermission` | ✅ Nouveau | Rien à faire |

---

## 4. Code mort identifié

### 4.1 Fichiers complètement morts (suppression safe)

| Fichier | Raison |
|---------|--------|
| `src/authorization/rbac.service.ts` | Jamais importé, jamais utilisé |
| `src/common/utils/resolve-user-scope.util.ts` | Jamais importé, jamais utilisé |

### 4.2 Fichiers morts quand les imports résidus seront supprimés

| Fichier | Condition de suppression |
|---------|------------------------|
| `src/authorization/casl.module.ts` | Quand les 4 modules retireront `CaslModule` de leurs imports |
| `src/authorization/casl-ability.factory.ts` | Même condition — plus aucun consommateur |
| `src/authorization/rbac.module.ts` | Quand les 4 modules retireront `RbacModule` de leurs imports |
| `src/authorization/rbac.types.ts` | Utilisé uniquement par `casl-ability.factory.ts` |
| Tout le dossier `src/authorization/` | Quand tout est nettoyé — suppression complète |

### 4.3 Code mort dans les utils

| Fichier | Problème |
|---------|---------|
| `resolve-attendee-scope.util.ts` | Retourne `'org'` comme scope, mais `:org` n'existe plus dans les permissions |
| `resolve-event-scope.util.ts` | Même problème — branche `:org` dans le code = dead code |

### 4.4 Problème `badge-generation.controller.ts`

Ce contrôleur utilise `@Permissions('badges.create:org', 'badges.create:any')`. Le scope `:org` n'existe plus dans les permissions seedées — ces endpoints risquent de ne jamais autoriser un ADMIN qui a `:any` à matcher `:org`.

---

## 5. Risques de l'architecture actuelle

### 5.1 Risques de sécurité 🔴

| # | Risque | Sévérité | Détail |
|---|--------|----------|--------|
| 1 | **Confusion sur quel système protège quoi** | 🔴 Haute | Un développeur pourrait croire que `CaslModule` protège un endpoint alors qu'il est mort — fausse sensation de sécurité |
| 2 | **Sessions sans authz** | 🔴 Haute | `sessions.controller.ts` n'a que `JwtAuthGuard` — n'importe quel user authentifié peut CRUD les sessions de n'importe quel event/org |
| 3 | **Scope `:org` fantôme** | 🟡 Moyenne | `badge-generation.controller.ts` utilise `badges.create:org` dans `@Permissions()` mais `:org` n'est pas dans le seeder → potentiel refus illégitime |
| 4 | **`permissions.length === 0 → 'any'`** | 🔴 Haute | Dans `RequirePermissionGuard`, si aucune permission n'est trouvée pour un user, le fallback enrichit avec scope `any` — un user sans permissions aurait accès total |
| 5 | **resolve-*-scope.util.ts retournent 'org'** | 🟡 Moyenne | Les services filtrent par scope `'org'` qui peut ne pas matcher les requêtes Prisma attendues |

### 5.2 Risques techniques 🟡

| # | Risque | Sévérité | Détail |
|---|--------|----------|--------|
| 6 | **Dette technique croissante** | 🟡 Moyenne | 8 imports résidus, 5+ fichiers morts — chaque nouveau dev perd du temps à comprendre quel système est actif |
| 7 | **Deux décorateurs pour la même chose** | 🟡 Moyenne | `@Permissions()` vs `@RequirePermission()` — risque de confusion lors de l'ajout de nouveaux endpoints |
| 8 | **Modules métier couplés à Prisma** | 🟡 Moyenne | Les services injectent directement `PrismaService` — impossible de tester unitairement sans mock DB complet |
| 9 | **Pas de tests d'autorisation** | 🔴 Haute | Aucun test ne vérifie que les guards refusent correctement les accès non autorisés |

### 5.3 Risques organisationnels

| # | Risque | Sévérité | Détail |
|---|--------|----------|--------|
| 10 | **Onboarding difficile** | 🟡 Moyenne | Un nouveau développeur trouve 3 systèmes auth, ne sait pas lequel utiliser |
| 11 | **Incohérence de patterns** | 🟡 Moyenne | `authz` est hexagonal, tous les modules métier sont plats NestJS → pas de convention claire |

---

## 6. Bénéfices du passage en full hexagonal

### 6.1 Pour les modules métier

| Bénéfice | Explication |
|----------|-------------|
| **Testabilité** | Le `core/` (service + use-cases) est testable sans DB, sans NestJS, sans HTTP. Tu mock les ports. |
| **Lisibilité** | La logique métier est isolée dans `core/`, l'infra dans `adapters/`. Un dev sait immédiatement où chercher. |
| **Évolutivité** | Changer de DB (Prisma → TypeORM, ou ajouter Redis cache) = modifier uniquement l'adapter, pas la logique métier. |
| **Cohérence** | Un seul pattern pour tout le backend — plus de confusion entre styles. |

### 6.2 Pour l'autorisation

| Bénéfice | Explication |
|----------|-------------|
| **Un seul système** | Supprimer `authorization/` entièrement → un seul endroit pour tout l'authz |
| **Un seul décorateur** | `@RequirePermission()` partout → zéro ambiguïté |
| **Sécurité vérifiable** | Avec des ports, on peut tester que le guard refuse correctement sans lancer l'app |

### 6.3 Ce que ça NE change PAS

- Les routes HTTP restent identiques
- Le schéma Prisma ne change pas
- Le frontend et le mobile ne sont pas impactés
- Les permissions dans le seeder restent les mêmes

---

## 7. Plan de migration

### Phase 0 — Nettoyage immédiat (30 min, 0 risque)

**Actions sans aucun impact fonctionnel :**

```
□ 1. Supprimer import CaslModule dans :
     - registrations.module.ts
     - email.module.ts
     - events.module.ts
     - partner-scans.module.ts

□ 2. Supprimer import RbacModule dans :
     - attendees.module.ts
     - organizations.module.ts
     - roles.module.ts
     - permissions.module.ts

□ 3. Supprimer les fichiers morts :
     - src/authorization/rbac.service.ts
     - src/common/utils/resolve-user-scope.util.ts

□ 4. Supprimer le dossier src/authorization/ entièrement
     (CaslModule est @Global mais personne ne consomme CaslAbilityFactory)

□ 5. Migrer attendee-types.controller.ts et badge-generation.controller.ts
     de @Permissions() + PermissionsGuard
     vers @RequirePermission() + RequirePermissionGuard

□ 6. Supprimer src/common/guards/permissions.guard.ts
     et src/common/decorators/permissions.decorator.ts
```

**Résultat : un seul système d'autorisation, un seul décorateur, zéro code mort.**

---

### Phase 1 — Migration `partner-scans` en hexagonal (1-2h)

Premier module migré comme proof-of-concept :

```
src/modules/partner-scans/
├── partner-scans.module.ts              ← câble ports ↔ adapters NestJS DI
│
├── core/                                ← ZÉRO import NestJS/Prisma
│   ├── types.ts                         ← CreateScanCommand, ScanResult, etc.
│   ├── partner-scans.service.ts         ← orchestre les use-cases
│   └── use-cases/
│       ├── create-partner-scan.ts       ← validation, anti-doublon, snapshot
│       ├── list-partner-scans.ts        ← filtrage par scope
│       ├── update-partner-scan.ts       ← ownership check
│       ├── delete-partner-scan.ts       ← ownership check
│       └── export-partner-scans.ts      ← génération CSV
│
├── ports/
│   ├── partner-scan.repository.port.ts  ← create, findById, findAll, update, delete
│   ├── event-access.port.ts             ← hasAccess(orgId, eventId, userId)
│   └── registration.port.ts             ← findById, getAttendeeSnapshot
│
└── adapters/
    ├── db/
    │   ├── prisma-partner-scan.repository.ts
    │   ├── prisma-event-access.adapter.ts
    │   └── prisma-registration.adapter.ts
    └── http/
        ├── partner-scans.controller.ts
        └── dto/
            ├── create-partner-scan.dto.ts
            ├── update-partner-scan.dto.ts
            └── list-partner-scans.dto.ts
```

---

### Phase 2 — Migration progressive des autres modules (1 module à la fois)

Ordre recommandé (du plus simple au plus complexe) :

| Priorité | Module | Complexité | Raison de l'ordre |
|----------|--------|-----------|-------------------|
| 1 | `tags` | 🟢 Simple | Petit module, peu de logique métier |
| 2 | `invitations` | 🟢 Simple | CRUD simple avec email |
| 3 | `attendee-types` | 🟢 Simple | CRUD pur |
| 4 | `badges` | 🟡 Moyen | Relations avec templates |
| 5 | `badge-templates` | 🟡 Moyen | Upload/storage |
| 6 | `email` | 🟡 Moyen | Templates + envoi |
| 7 | `users` | 🟡 Moyen | Lié aux rôles |
| 8 | `organizations` | 🟡 Moyen | Multi-tenant core |
| 9 | `roles` + `permissions` | 🟡 Moyen | Liés entre eux |
| 10 | `attendees` | 🔴 Complexe | Import/export, soft delete |
| 11 | `events` | 🔴 Complexe | Module le plus gros, relations multiples |
| 12 | `registrations` | 🔴 Complexe | Le plus gros, check-in/out, badges, import |
| 13 | `sessions` | 🟢 Simple | Mais faut d'abord ajouter l'authz |
| 14 | `badge-generation` | 🟡 Moyen | PDF generation + storage |

**Règle : un module à la fois, on teste, on merge, on passe au suivant.**

---

### Phase 3 — Corrections de sécurité (À faire ASAP, indépendant de la migration)

```
□ 1. Ajouter des permissions à sessions.controller.ts
     (actuellement JwtAuthGuard seul = n'importe quel user auth peut CRUD)

□ 2. Corriger le fallback `permissions.length === 0 → 'any'`
     dans RequirePermissionGuard → devrait REFUSER, pas donner scope 'any'

□ 3. Corriger badge-generation.controller.ts :
     remplacer 'badges.create:org' par 'badges.create:any'
     (le scope :org n'existe plus)

□ 4. Nettoyer resolve-attendee-scope.util.ts et resolve-event-scope.util.ts :
     remplacer les branches `:org` par `:any`
```

---

## Annexe A — Où `authorization/` est utilisé (imports exacts)

```typescript
// --- Imports CaslModule ---
// src/modules/registrations/registrations.module.ts
import { CaslModule } from '../../authorization/casl.module';
// → imports: [PrismaModule, CaslModule, BadgeGenerationModule, EmailModule]

// src/modules/email/email.module.ts
import { CaslModule } from '../../authorization/casl.module';

// src/modules/events/events.module.ts
import { CaslModule } from '../../authorization/casl.module';

// src/modules/partner-scans/partner-scans.module.ts
import { CaslModule } from '../../authorization/casl.module';

// --- Imports RbacModule ---
// src/modules/attendees/attendees.module.ts
import { RbacModule } from '../../authorization/rbac.module';

// src/modules/organizations/organizations.module.ts
import { RbacModule } from '../../authorization/rbac.module';

// src/modules/roles/roles.module.ts
import { RbacModule } from '../../authorization/rbac.module';

// src/modules/permissions/permissions.module.ts
import { RbacModule } from '../../authorization/rbac.module';
```

**Aucun de ces modules n'injecte `CaslAbilityFactory` dans un controller ou service.**  
→ **Tous ces imports sont supprimables immédiatement.**

---

## Annexe B — Comment `platform/authz` est câblé (ref rapide)

```typescript
// authz.module.ts — Injection hexagonale
{ provide: RBAC_QUERY_PORT,    useClass: PrismaRbacQueryAdapter }      // Port → Adapter
{ provide: MEMBERSHIP_PORT,    useClass: PrismaMembershipAdapter }
{ provide: AUTH_CONTEXT_PORT,  useClass: PrismaAuthContextAdapter }
{ provide: MODULE_GATING_PORT, useClass: PrismaModuleGatingAdapter }

// Factory pour les services core (sans dépendance NestJS)
{ provide: PermissionResolver,
  useFactory: (rbacQueryPort) => new PermissionResolver(rbacQueryPort),
  inject: [RBAC_QUERY_PORT] }

{ provide: AuthorizationService,
  useFactory: (pr, mp) => new AuthorizationService(pr, mp),
  inject: [PermissionResolver, MEMBERSHIP_PORT] }
```

→ Les services `core/` n'ont **jamais** de `@Injectable()`.  
→ Ils sont construits via `useFactory` pour rester purement TypeScript.

---

## Annexe C — Différence entre `@RequirePermission()` et `@Permissions()`

| Aspect | `@RequirePermission()` | `@Permissions()` |
|--------|----------------------|------------------|
| Source | `platform/authz/adapters/http/decorators/` | `common/decorators/permissions.decorator.ts` |
| Guard | `RequirePermissionGuard` | `PermissionsGuard` |
| Moteur RBAC | `AuthorizationService` (nouveau) | `AuthorizationService` (nouveau aussi) |
| Sémantique | 1 permission (AND avec `@RequireAllPermissions`) | Plusieurs permissions en OR |
| Nb contrôleurs | 13 | 2 (`attendee-types`, `badge-generation`) |
| Recommandation | ✅ Standard du projet | ⚠️ À migrer vers `@RequirePermission` |

**Les deux utilisent le même moteur.** La migration de `@Permissions` → `@RequirePermission` est purement syntaxique, zéro risque fonctionnel.
