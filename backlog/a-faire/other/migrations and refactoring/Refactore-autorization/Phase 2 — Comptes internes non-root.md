# Phase 2 — Comptes internes non-root

> **Statut** : conception uniquement. Ne sera implémenté qu'après stabilisation complète de la V1.
>
> **Principe** : les users internes ne sont pas des users tenant. Ils opèrent *sur* le produit, pas *dans* le produit.

---

## Principe cible — 3 couches distinctes

```
┌─────────────────────────────────────────────────────────────┐
│                          User                                │
│                    (identité unique)                          │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │  Tenant access   │  │ Internal access   │  │   System   │ │
│  │                  │  │                   │  │   access   │ │
│  │  OrgUser         │  │ InternalUserRole  │  │            │ │
│  │  TenantUserRole  │  │ InternalOrgAccess │  │ UserSystem │ │
│  │  EventAccess     │  │                   │  │ Access     │ │
│  │                  │  │                   │  │            │ │
│  │  → membre client │  │ → staff interne   │  │ → root     │ │
│  │  → rôle métier   │  │ → accès contrôlé  │  │ → bypass   │ │
│  │  → dans l'org    │  │ → sur des orgs    │  │ → exception│ │
│  └──────────────────┘  └──────────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

| Couche | Qui | Accès | Modélisation |
|--------|-----|-------|--------------|
| **Tenant** | Clients, utilisateurs métier | Membre de l'org, rôle tenant, permissions métier | `OrgUser` + `TenantUserRole` + `EventAccess` |
| **Internal** | Support, ops, onboarding, billing | Rôle interne + orgs assignées ou global | `InternalUserRole` + `InternalOrgAccess` |
| **System** | Root uniquement | Bypass total, très rare | `UserSystemAccess` |

### Le point clé

Les comptes internes non-root **ne doivent pas** être modélisés comme des rôles tenant.

Pourquoi ?
- Un support interne n'est pas un vrai membre client de l'org
- Il est un acteur interne avec un accès contrôlé sur cette org
- Mélanger les deux brouille : les permissions, l'audit, les invitations, les exports, la logique "qui appartient à l'organisation"

---

## Architecture cible

```
User
├── OrgUser --------------------------> appartenance client
├── TenantUserRole -------------------> rôle métier dans l'org
├── EventAccess ----------------------> ressources assignées (scope assigned)
├── InternalUserRole -----------------> rôle staff interne
├── InternalOrgAccess ----------------> orgs accessibles (staff interne limité)
└── UserSystemAccess -----------------> root / accès système exceptionnel
```

---

## Modèles Prisma

### 1. User (inchangé, étendu avec les nouvelles relations)

```prisma
model User {
  id                String   @id @default(uuid()) @db.Uuid
  email             String   @unique @db.Citext
  password_hash     String
  is_active         Boolean  @default(true)
  first_name        String?
  last_name         String?
  created_at        DateTime @default(now())
  updated_at        DateTime @updatedAt

  // Tenant access (V1 — existant)
  orgMemberships    OrgUser[]
  tenantRoles       TenantUserRole[]

  // Internal access (V2 — nouveau)
  internalRole      InternalUserRole?
  internalOrgAccess InternalOrgAccess[]

  // System access (V1 — existant)
  systemAccess      UserSystemAccess?

  @@map("users")
}
```

### 2. Rôles internes

> Table dédiée. On ne réutilise **pas** la table `Role` tenant pour éviter de remettre dans le même sac rôles tenant métier, rôles internes staff, et privilèges système.

```prisma
enum InternalAccessLevel {
  GLOBAL    // Accès à toutes les orgs (ex: OPS)
  LIMITED   // Accès seulement aux orgs assignées (ex: SUPPORT_AGENT)
}

model InternalRole {
  id            String              @id @default(uuid()) @db.Uuid
  code          String              @unique    // SUPPORT_AGENT, OPS, etc.
  name          String
  description   String?
  access_level  InternalAccessLevel @default(LIMITED)
  created_at    DateTime            @default(now())
  updated_at    DateTime            @updatedAt

  permissions   InternalRolePermission[]
  users         InternalUserRole[]

  @@map("internal_roles")
}
```

**Exemples de rôles** :

| Code | Nom | Access Level | Usage |
|------|-----|--------------|-------|
| `SUPPORT_AGENT` | Agent support | LIMITED | Support client, orgs assignées |
| `SUPPORT_MANAGER` | Manager support | GLOBAL | Supervise le support, toutes orgs |
| `OPS` | Opérations | GLOBAL | Gestion technique cross-org |
| `ONBOARDING_SPECIALIST` | Spécialiste onboarding | LIMITED | Setup des nouvelles orgs |
| `BILLING_ADMIN` | Admin facturation | LIMITED | Gestion billing par org |
| `READONLY_AUDITOR` | Auditeur lecture seule | LIMITED | Audit / conformité |

### 3. Permissions internes

> Séparées des permissions tenant. Nomenclature `platform.*` pour éviter toute confusion.

```prisma
model InternalPermission {
  id          String   @id @default(uuid()) @db.Uuid
  code        String   @unique    // platform.orgs.read, platform.users.reset_password, etc.
  name        String
  description String?
  created_at  DateTime @default(now())
  updated_at  DateTime @updatedAt

  rolePermissions InternalRolePermission[]

  @@map("internal_permissions")
}

model InternalRolePermission {
  role_id       String @db.Uuid
  permission_id String @db.Uuid

  role          InternalRole       @relation(fields: [role_id], references: [id], onDelete: Cascade)
  permission    InternalPermission @relation(fields: [permission_id], references: [id], onDelete: Cascade)

  @@id([role_id, permission_id])
  @@map("internal_role_permissions")
}
```

**Exemples de permissions** :

| Code | Description |
|------|-------------|
| `platform.orgs.read` | Lister/voir les organisations |
| `platform.orgs.update` | Modifier les paramètres d'une org |
| `platform.orgs.setup` | Configurer une nouvelle org |
| `platform.users.read` | Voir les users d'une org |
| `platform.users.invite` | Inviter des users |
| `platform.users.reset_password` | Reset de mot de passe |
| `platform.events.read` | Voir les events d'une org |
| `platform.events.manage` | Gérer les events |
| `platform.registrations.read` | Voir les inscriptions |
| `platform.billing.read` | Voir la facturation |
| `platform.billing.manage` | Gérer la facturation |
| `platform.printing.manage` | Gérer l'impression |
| `platform.support.access` | Accès support général |

### 4. Attribution du rôle interne à un user

```prisma
model InternalUserRole {
  user_id      String    @id @db.Uuid
  role_id      String    @db.Uuid
  assigned_by  String?   @db.Uuid       // Qui a assigné
  assigned_at  DateTime  @default(now())
  expires_at   DateTime?                 // Accès temporaire possible
  revoked_at   DateTime?                 // Révocation propre

  user         User         @relation(fields: [user_id], references: [id], onDelete: Cascade)
  role         InternalRole @relation(fields: [role_id], references: [id], onDelete: Cascade)

  @@index([role_id])
  @@map("internal_user_roles")
}
```

> Un seul rôle interne par user pour rester simple. Extensible plus tard si besoin.

### 5. Périmètre d'accès aux orgs

> C'est la table qui fait la vraie différence entre **rôle** (ce que tu peux faire) et **périmètre** (où tu peux le faire).

```prisma
model InternalOrgAccess {
  user_id      String    @db.Uuid
  org_id       String    @db.Uuid
  granted_by   String?   @db.Uuid       // Qui a donné l'accès
  granted_at   DateTime  @default(now())
  expires_at   DateTime?                 // Accès temporaire
  revoked_at   DateTime?                 // Révocation propre
  reason       String?                   // Justification auditée

  user         User         @relation(fields: [user_id], references: [id], onDelete: Cascade)
  organization Organization @relation(fields: [org_id], references: [id], onDelete: Cascade)

  @@id([user_id, org_id])
  @@index([org_id])
  @@map("internal_org_access")
}
```

**Règles d'accès** :

| Access Level du rôle | InternalOrgAccess | Résultat |
|---------------------|-------------------|----------|
| `GLOBAL` | Non requis | Accès à toutes les orgs |
| `LIMITED` | Requis | Accès seulement aux orgs listées dans la table |
| `LIMITED` | Aucune entrée | Aucun accès à aucune org |

---

## Pipeline d'autorisation futur

### Cas 1 — User tenant normal (inchangé V1)

```
→ Charger activeOrgId depuis le JWT
→ Charger TenantUserRole pour (userId, orgId)
→ Résoudre permissions tenant via RolePermission
→ Appliquer EventAccess si scope = assigned
→ Exécuter
```

### Cas 2 — User interne non-root

```
→ Charger InternalUserRole pour userId
→ Résoudre permissions internes via InternalRolePermission
→ Vérifier le périmètre :
   ├── Si rôle GLOBAL → accès à toutes les orgs
   └── Si rôle LIMITED → vérifier InternalOrgAccess pour (userId, activeOrgId)
→ L'interne DOIT avoir un activeOrgId quand il consulte/modifie des données tenant
→ Exécuter
```

### Cas 3 — Root (inchangé V1)

```
→ Bypass permissions
→ MAIS activeOrgId obligatoire pour les données tenant
→ Audit complet
→ Exécuter
```

### Identification du type d'acteur

```
Pour chaque requête authentifiée :

1. user.systemAccess?.is_root = true  → actor_context = root
2. user.internalRole exists           → actor_context = internal
3. user.orgMemberships exists         → actor_context = tenant
4. Aucun                              → 403

Le actor_context détermine QUEL pipeline d'autorisation est utilisé.
```

---

## Exemples métier concrets

### Exemple 1 — Support agent limité

**Sonia** est `SUPPORT_AGENT` (LIMITED).

Permissions :
- `platform.orgs.read`
- `platform.events.read`
- `platform.registrations.read`

Accès via `InternalOrgAccess` :
- ORG_A ✅
- ORG_C ✅

Conséquence :
- Elle peut ouvrir ORG_A et ORG_C
- Elle **ne peut pas** ouvrir ORG_B
- Elle **ne peut pas** gérer la facturation (pas la permission)
- Elle **ne peut pas** modifier les events (pas `platform.events.manage`)

### Exemple 2 — Onboarding specialist

**Karim** est `ONBOARDING_SPECIALIST` (LIMITED).

Permissions :
- `platform.orgs.read`
- `platform.orgs.setup`
- `platform.users.invite`

Accès : seulement `ORG_NEW_CLIENT`

Il peut :
- Configurer l'org
- Inviter les premiers admins
- Voir certaines données de setup

Il **ne peut pas** :
- Toucher les exports GDPR
- Accéder aux autres orgs
- Gérer la facturation

### Exemple 3 — Ops global

**Lina** est `OPS` (GLOBAL).

Permissions :
- `platform.orgs.read`
- `platform.events.read`
- `platform.events.manage`
- `platform.printing.manage`

**Accès global** — elle peut intervenir sur toutes les orgs.

**Différence avec root** :
- Elle reste limitée à ses permissions internes
- Elle n'a **pas** de bypass absolu
- Elle ne peut pas tout faire (pas de `platform.billing.*`, pas de `platform.users.reset_password`)

---

## Audit — obligatoire pour cette couche

Cette couche interne doit être **très auditée**. Le support interne sans audit est un trou noir.

| Action | Données tracées |
|--------|----------------|
| Attribution/retrait de rôle interne | userId, targetUserId, roleCode, by, at |
| Attribution/retrait d'accès org | userId, targetUserId, orgId, by, at, reason |
| Changement d'org active | userId, fromOrgId, toOrgId |
| Consultation de données sensibles | userId, orgId, resource, action |
| Export | userId, orgId, type d'export |
| Reset password | userId, targetUserId, orgId |
| Modifications importantes | userId, orgId, resource, before, after |
| Impersonation (si ajouté un jour) | userId, targetUserId, orgId, duration |

---

## Ce qu'il ne faut PAS faire

| Anti-pattern | Pourquoi c'est dangereux |
|-------------|-------------------------|
| Réutiliser `TenantUserRole` pour le support interne | Crée un faux user métier dans l'org |
| Créer de faux `OrgUser` internes juste pour "voir" une org | Brouille l'appartenance réelle à l'org |
| Mettre support/root/internal dans la même table `Role` | Mélange 3 réalités différentes |
| Utiliser root comme solution de confort quotidienne | Root = exception, pas outil de travail |
| Donner accès global par défaut à tous les comptes internes | Principe du moindre privilège violé |

---

## Plan de migration futur

> Chaque étape est indépendante et peut être livrée séparément.

```
Étape 1 — Garder V1 tenant-only intacte
   └─ Rien ne casse. V1 continue de fonctionner.

Étape 2 — Ajouter les tables internes (sans brancher le runtime)
   └─ internal_roles
   └─ internal_permissions
   └─ internal_role_permissions
   └─ internal_user_roles
   └─ internal_org_access
   └─ Seed des rôles et permissions de base

Étape 3 — Back-office interne minimal
   └─ Lister les orgs accessibles
   └─ Choisir une org active
   └─ Lire quelques données (read-only)

Étape 4 — Brancher l'autorisation interne
   └─ Sur quelques endpoints safe en lecture seule
   └─ actor_context = internal dans le pipeline

Étape 5 — Étendre à l'écriture
   └─ Progressivement, si le métier le justifie
   └─ Chaque nouveau endpoint = nouvelle permission explicite

Étape 6 — Impersonation contrôlée (optionnel)
   └─ Seulement si vraiment nécessaire
   └─ Audit maximal + durée limitée
```

---

## Résumé

> **Le vrai principe à retenir :**
>
> - Les **tenant users** appartiennent au produit
> - Les **internal users** opèrent sur le produit
> - Le **root** contrôle le système
>
> Ces 3 réalités ne doivent pas partager la même modélisation.

---

## Annexe — Enregistrement global de `TenantContextGuard` (Phase 2)

> **Statut** : à faire après stabilisation de la V1.
>
> **Contexte** : depuis le fix C1, `TenantContextGuard` est correct (il ne lit plus `user.mode`) et supporte le décorateur opt-out `@SkipTenantContext()`. Il est cependant déclaré nulle part — les controllers doivent donc l'ajouter manuellement via `@UseGuards()`. Ce n'est pas robuste : un contrôleur oublié = un trou.

### Objectif

Faire en sorte que **toute requête authentifiée** soit obligée d'avoir un `currentOrgId`, sauf opt-out explicite. Le tenant-context devient une garantie d'infrastructure, pas une discipline de développeur.

### Pattern NestJS — `APP_GUARD`

NestJS permet d'enregistrer un guard une seule fois en global via le token `APP_GUARD`. Tous les controllers en héritent automatiquement.

```ts
// src/app.module.ts
import { APP_GUARD } from '@nestjs/core';
import { TenantContextGuard } from './common/guards/tenant-context.guard';

@Module({
  // ...
  providers: [
    // ... existants
    {
      provide: APP_GUARD,
      useClass: TenantContextGuard,
    },
  ],
})
export class AppModule {}
```

Important : il faut que `JwtAuthGuard` soit déclenché **avant** (pour avoir `req.user`). Comme `JwtAuthGuard` est déjà appliqué au niveau de chaque controller authentifié, l'ordre actuel fonctionne. Si on globalise aussi `JwtAuthGuard` un jour, l'ordre d'enregistrement dans `providers[]` détermine l'ordre d'exécution.

### Routes opt-out à marquer

Les routes suivantes n'ont pas (encore) de `currentOrgId` quand elles sont appelées et doivent être annotées `@SkipTenantContext()` (ou `@Public()` si elles sont déjà publiques) :

| Route | Pourquoi |
|-------|----------|
| `POST /auth/login` | Pas encore authentifié |
| `POST /auth/refresh` | Le refresh ne dépend pas d'une org |
| `GET /auth/me/orgs` | L'utilisateur choisit son org après cet appel |
| `POST /auth/switch-org` | C'est l'appel qui crée le contexte |
| `GET /auth/policy` | Métadonnée globale |
| Tous les endpoints publics (déjà `@Public()`) | Aucun user |
| Health checks | Aucun user |

### Plan de déploiement

1. **Étape A — audit préalable** : grep tous les controllers, lister ceux qui n'ont **pas** besoin de `currentOrgId`. Annoter avec `@SkipTenantContext()`.
2. **Étape B — activation derrière feature-flag** : enregistrer `APP_GUARD` mais derrière un flag d'env `ENFORCE_TENANT_CONTEXT=true`. Tester en staging.
3. **Étape C — observation** : monitorer les 400 (`No organization context`). Identifier les routes oubliées. Annoter ou corriger.
4. **Étape D — activation prod** : flag à `true`. Le guard devient bloquant pour tout le monde.
5. **Étape E — nettoyage** : retirer les `@UseGuards(TenantContextGuard)` manuels devenus redondants.

### Risques

- **Faux positifs** : si un controller fait un travail légitime sans org, il sera bloqué. D'où l'étape B/C avec feature-flag.
- **Tests** : tous les tests E2E qui n'utilisent pas un JWT avec `currentOrgId` casseront. Prévoir un helper de test qui injecte un payload complet.

### Critère de succès

- 0 `req.user.currentOrgId` lu sans garantie qu'il existe
- Les nouveaux controllers sont tenant-safe par défaut, sans rien faire
- Une seule façon d'opter-out (`@SkipTenantContext()`), traçable par grep

---


> **Pour la suite (items hors-scope V1, dette V1, suggestions V2 transverses) → voir** [Refactor-Authorization-V2.md](./Refactor-Authorization-V2.md).
