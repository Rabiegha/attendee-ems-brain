# Analyse réelle du système d'autorisation — État actuel du code

> Document généré à partir d'une **lecture directe du code** (pas de la doc).
> Branche : `Refactore-authorization-v1`
> Backend : `attendee-ems-back/src/`

---

## 0. TL;DR

| Question | Réponse courte |
|----------|----------------|
| Le check `isRoot` leak-t-il dans les services métier ? | **Non**. Centralisé dans `AuthorizationService`, `ScopeEvaluator`, `verifyOrgAccess`. ✅ |
| Le check `mode === 'platform'` leak-t-il ? | **Non** (même logique que isRoot). ✅ |
| Le système actuel est-il hexagonal ? | **Partiellement**. `src/platform/authz/` l'est, les `src/modules/*` ne le sont pas. |
| Combien de modules à migrer pour aller hexagonal ? | **~10–12 modules** (sur ~21 au total) |
| L'architecture actuelle est-elle "bonne" ? | **Correcte mais incohérente**. Le RBAC est propre, le métier est NestJS classique. Le scope logic est éparpillé (3 endroits) — c'est le vrai problème. |
| Est-ce que je peux ajouter d'autres axes (ex: tags, projets) ? | **Oui**, mais ça va aggraver l'éparpillement actuel tant que le scope n'est pas centralisé. |

---

## 1. Comment marchent les permissions aujourd'hui (le vrai flow)

### 1.1 Décorateurs sur un controller

```ts
@UseGuards(JwtAuthGuard, RequirePermissionGuard)
@RequirePermission('attendees.read')
@Get()
findAll(...) { ... }
```

### 1.2 Ordre d'exécution

```
HTTP Request
   │
   ▼
JwtAuthGuard                           ← valide le JWT
   │  → req.user = JwtPayload minimal
   │     { sub, mode, currentOrgId }
   ▼
RequirePermissionGuard                 ← le vrai cerveau
   │  1. Lit @RequirePermission('xxx') du contexte
   │  2. Construit AuthContext via PrismaAuthContextAdapter
   │     → enrichit avec isRoot, isPlatform depuis la DB
   │  3. Appelle AuthorizationService.can()
   │  4. Si Decision.allow → laisse passer
   │     Sinon → 403
   ▼
Controller → Service → Prisma
```

Fichiers clés :
- [src/common/guards/jwt-auth.guard.ts](../../../../../../attendee-ems-back/src/common/guards/jwt-auth.guard.ts)
- [src/platform/authz/adapters/http/guards/require-permission.guard.ts](../../../../../../attendee-ems-back/src/platform/authz/adapters/http/guards/require-permission.guard.ts)

### 1.3 Contenu du `req.user`

Après JWT seul (avant le guard d'authz) :

```ts
{
  sub: "user-id",
  id: "user-id",                       // alias compat
  mode: 'tenant' | 'platform',
  currentOrgId: "org-id" | null,
  iat, exp
}
```

⚠️ Le JWT **ne contient pas** les permissions. C'est `RequirePermissionGuard` qui les charge depuis la DB et les ajoute via `AuthContext`.

### 1.4 Construction de `AuthContext`

Dans [src/platform/authz/adapters/db/prisma-auth-context.adapter.ts](../../../../../../attendee-ems-back/src/platform/authz/adapters/db/prisma-auth-context.adapter.ts) :

```ts
async buildAuthContext(jwt): Promise<AuthContext> {
  const platformRole = await prisma.platformUserRole.findUnique({
    where: { user_id: jwt.sub },
    include: { role: true },
  });

  return {
    userId: jwt.sub,
    mode: jwt.mode,
    isPlatform: !!platformRole,
    isRoot: platformRole?.role.is_root ?? false,   // ← ICI le seul endroit où isRoot vient de la DB
    currentOrgId: jwt.currentOrgId ?? null,
  };
}
```

### 1.5 Décision d'autorisation

[src/platform/authz/core/authorization.service.ts](../../../../../../attendee-ems-back/src/platform/authz/core/authorization.service.ts) :

```ts
async can(permissionKey, authContext, rbacContext): Promise<Decision> {
  // 1. Root bypass total
  if (authContext.isRoot) return Decisions.allow();

  // 2. Vérifs contextuelles (membership org / platform org access)
  const ctx = await this.checkContext(authContext, rbacContext);
  if (!ctx.allowed) return ctx;

  // 3. Charger les grants (rôle → permissions)
  const { grants } = await this.permissionResolver.resolve(authContext);

  // 4. Trouver le grant correspondant
  const grant = this.permissionResolver.findGrant(grants, permissionKey);
  if (!grant) return Decisions.denyNoPermission(permissionKey);

  // 5. Évaluer le scope (any/assigned/own)
  const ok = ScopeEvaluator.evaluate(grant.scope, authContext, rbacContext);
  return ok ? Decisions.allow() : Decisions.denyScopeMismatch(...);
}
```

### 1.6 Le scope (`any` / `assigned` / `own`)

[src/platform/authz/core/scope-evaluator.ts](../../../../../../attendee-ems-back/src/platform/authz/core/scope-evaluator.ts) :

```ts
switch (requiredScope) {
  case 'any':      return true;
  case 'assigned': return assignedUserIds?.includes(authContext.userId) ?? true;
  case 'own':      return resourceOwnerId === authContext.userId;
}
```

**⚠️ Problème majeur** : le scope est évalué **à 3 endroits différents** (voir §3.1).

### 1.7 `verifyOrgAccess()`

[src/auth/auth.service.ts](../../../../../../attendee-ems-back/src/auth/auth.service.ts) (autour de la ligne 542) :

```ts
private async verifyOrgAccess(userId, orgId): Promise<boolean> {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { orgMemberships, platformRole: { include: { role } }, platformOrgAccess },
  });

  if (user.orgMemberships.some(m => m.org_id === orgId)) return true;
  if (user.platformRole) {
    if (user.platformRole.role.is_root) return true;     // root = toutes les orgs
    return user.platformOrgAccess.some(a => a.org_id === orgId);
  }
  return false;
}
```

Utilisé dans `/auth/switch-org` et `/auth/token-for-org`.

### 1.8 CASL — où il en est

[src/authorization/casl-ability.factory.ts](../../../../../../attendee-ems-back/src/authorization/casl-ability.factory.ts) :

```ts
if (user.role === 'SUPER_ADMIN') can(Action.Manage, 'all');
else if (user.role === 'ADMIN')  can(Action.Manage, 'all');
```

**CASL n'est PAS le moteur de décision serveur.** Il sert seulement à exposer un snapshot via `GET /auth/me/ability` pour le front.

### 1.9 Endpoint type "policy snapshot"

`GET /auth/policy` **n'existe pas**. À la place :
- [src/platform/authz/adapters/http/controllers/me-ability.controller.ts](../../../../../../attendee-ems-back/src/platform/authz/adapters/http/controllers/me-ability.controller.ts) → `GET /auth/me/ability` retourne les grants + `isRoot`.

---

## 2. Le check `isRoot` — Est-il bien isolé ?

### 2.1 Liste exhaustive des occurrences

| Fichier | Type d'usage | OK ? |
|---------|--------------|------|
| [src/platform/authz/core/authorization.service.ts](../../../../../../attendee-ems-back/src/platform/authz/core/authorization.service.ts) | Bypass dans `can()` | ✅ Centre |
| [src/platform/authz/core/scope-evaluator.ts](../../../../../../attendee-ems-back/src/platform/authz/core/scope-evaluator.ts) | Bypass scope | ✅ Centre |
| [src/platform/authz/core/types.ts](../../../../../../attendee-ems-back/src/platform/authz/core/types.ts) | Définition du type | ✅ Type |
| [src/platform/authz/adapters/db/prisma-auth-context.adapter.ts](../../../../../../attendee-ems-back/src/platform/authz/adapters/db/prisma-auth-context.adapter.ts) | Lecture DB → AuthContext | ✅ Adapter |
| [src/platform/authz/adapters/http/controllers/me-ability.controller.ts](../../../../../../attendee-ems-back/src/platform/authz/adapters/http/controllers/me-ability.controller.ts) | Expose au front | ✅ Snapshot |
| [src/auth/auth.service.ts](../../../../../../attendee-ems-back/src/auth/auth.service.ts) | Login response + verifyOrgAccess | ✅ Auth |
| `src/modules/**` | **Aucune occurrence** | ✅✅ |

### 2.2 Verdict

> **Le check `isRoot` est correctement centralisé.** Aucun `if (user.isRoot)` ne pollue les services métier. C'est conforme à la règle d'or énoncée dans la doc v1.

Même chose pour `mode === 'platform'` : présent uniquement dans le core authz et dans `auth.service.ts`. Pas de leak métier.

---

## 3. Les vrais problèmes architecturaux

### 3.1 ⚠️ CRITIQUE — Le scope est évalué à 3 endroits

C'est **le** problème principal du système actuel.

| Endroit | Code | Rôle |
|---------|------|------|
| **Core RBAC** | `ScopeEvaluator.evaluate()` | Vérifie si le user a le droit (binaire) |
| **Utility** | [src/common/utils/resolve-event-scope.util.ts](../../../../../../attendee-ems-back/src/common/utils/resolve-event-scope.util.ts) | Devine le scope effectif depuis `user.permissions[]` (chaînes) |
| **Service métier** | `attendees.service.ts`, `events.service.ts` | Applique le filtre Prisma (`where`) |

Conséquence : trois sources de vérité pour la même règle. Le jour où tu ajoutes un scope (ex: `org`, `tenant`), tu dois le faire à 3 endroits → bugs garantis.

### 3.2 Pas de couche Repository

```ts
// attendees.service.ts
async findAll(dto, ctx) {
  const where = {};
  // ... métier mélangé avec Prisma where
  return this.prisma.attendee.findMany({ where, skip, take });
}
```

- Métier + accès données dans le même fichier
- Vendor lock-in Prisma
- Tests unitaires lourds (besoin de mocker Prisma)

### 3.3 Multi-tenancy non systématique

L'isolation `org_id` se fait **manuellement** dans chaque service via `resolveEffectiveOrgId(...)`. Pas de Prisma middleware ni de guard global qui force l'injection.

→ **Risque** : un dev oublie le filtre `where: { org_id }` et expose des données cross-tenant.

### 3.4 CASL côté serveur = mort

Le `CaslAbilityFactory` retourne des abilities qui **ne sont jamais checkées côté serveur**. C'est de la dette : soit on s'en sert vraiment, soit on supprime.

### 3.5 Deux systèmes RBAC parallèles

- **Ancien** : `src/authorization/` (CASL + RbacService legacy)
- **Nouveau** : `src/platform/authz/` (hexagonal, ports/adapters)

Les deux existent en même temps. Le refactoring n'est pas terminé.

---

## 4. Architecture actuelle — Schéma

```
src/
├── auth/                          ← Login, JWT, switch-org
├── authorization/                 ← LEGACY CASL (à supprimer ou réintégrer)
│   ├── casl-ability.factory.ts
│   ├── rbac.service.ts
│   └── rbac.types.ts
├── common/
│   ├── guards/
│   │   ├── jwt-auth.guard.ts          ✅
│   │   ├── permissions.guard.ts       ⚠️  legacy
│   │   └── tenant-context.guard.ts
│   └── utils/
│       └── resolve-*-scope.util.ts    ⚠️  scope fragmenté
├── platform/
│   └── authz/                     ✅ HEXAGONAL propre
│       ├── core/                  ← Pure (pas de Prisma)
│       │   ├── authorization.service.ts
│       │   ├── permission-resolver.ts
│       │   ├── scope-evaluator.ts
│       │   └── types.ts
│       ├── ports/                 ← Interfaces
│       └── adapters/
│           ├── db/                ← Prisma
│           └── http/              ← Guards + Controllers
└── modules/                       ❌ NestJS classique
    ├── attendees/  events/  registrations/
    ├── users/      organizations/  roles/
    ├── invitations/  badges/  …
    └── (≈ 21 modules au total)
```

---

## 5. Peut-on ajouter d'autres axes d'autorisation ?

Axes existants :

| Axe | Présent ? | Comment |
|-----|-----------|---------|
| Permission par rôle | ✅ | `RolePermission` |
| Scope `any/assigned/own` | ✅ | `Permission.scope` + `ScopeEvaluator` |
| Filtrage par `org_id` | ✅ | Manuel dans services |
| Filtrage par `event_access` | ✅ | Pour scope `assigned` |
| Filtrage par tags / projets / départements | ❌ | N'existe pas |
| Conditions dynamiques (ex: `event.status === 'published'`) | ❌ | N'existe pas |

**Réponse honnête** : techniquement oui, tu peux ajouter d'autres axes. Mais tant que le **scope n'est pas centralisé** (§3.1), chaque nouvel axe va être implémenté à 3 endroits différents. C'est un effet boule de neige.

**Recommandation** : avant d'ajouter un axe, **centraliser** la résolution du scope/filtre dans une seule couche (par exemple un `AuthorizationContext` qui produit un `PrismaWhereInput`).

---

## 6. Migration vers Hexagonal — Effort

### 6.1 Inventaire des modules à migrer

Modules dans `src/modules/` :

| Module | Taille | Migration |
|--------|--------|-----------|
| attendees | Moyen | 🔴 Oui (cœur métier) |
| registrations | Grand | 🔴 Oui (cœur métier) |
| events | Grand | 🔴 Oui (cœur métier) |
| organizations | Moyen | 🔴 Oui |
| users | Moyen | 🔴 Oui |
| roles | Petit | 🟠 Oui (lié à authz) |
| invitations | Petit | 🟠 Oui |
| badges | Petit | 🟠 Oui |
| badge-templates | Petit | 🟠 Oui |
| badge-generation | Moyen | 🟠 Oui |
| companies | Petit | 🟠 Oui |
| event-tables | Petit | 🟡 Optionnel |
| sessions | Petit | 🟡 Optionnel |
| tags | Petit | 🟡 Optionnel |
| attendee-types | Petit | 🟡 Optionnel |
| email | Petit | 🟢 Non |
| email-templates | Petit | 🟢 Non |
| n8n | Petit | 🟢 Non (intégration) |
| partner-scans | Petit | 🟢 Non |
| permissions | Petit | 🟢 Non (CRUD admin) |
| public | Petit | 🟢 Non |

**Bilan** :
- 🔴 **5 modules cœur** = effort élevé
- 🟠 **6 modules secondaires** = effort moyen
- 🟡 4 modules optionnels
- 🟢 6 modules à laisser tels quels

→ **10 à 12 modules** à migrer pour une cohérence raisonnable.

### 6.2 Effort par module (template hexagonal)

Pour un module type :
1. Créer `domain/` (entités + value objects + ports/repository interfaces) — ~petit
2. Créer `application/` (use cases) — ~moyen, c'est ici que tu déplaces la logique des services actuels
3. Créer `infrastructure/` (Prisma repository implementant le port) — ~petit
4. Adapter le controller pour appeler les use cases — ~petit
5. Tests unitaires des use cases (sans Prisma) — ~bonus important

**Estimation grossière** : 1 à 3 jours par module métier selon complexité.

### 6.3 Faut-il vraiment migrer ?

**Mon avis honnête** :

| Si tu fais... | Alors hexagonal vaut... |
|---------------|--------------------------|
| Beaucoup de tests unitaires | ✅ Oui |
| Tu prévois de changer Prisma un jour | ✅ Oui |
| Tu veux du CQRS / event sourcing | ✅ Oui |
| Ton équipe grandit (>5 devs) | ✅ Oui |
| Tu es 1–2 devs, pas de tests, vélocité prio | ❌ Probablement non |

**Pour ton cas (équipe petite, produit en croissance)** : la priorité réelle n'est pas hexagonal. C'est :

1. **Centraliser le scope/filter** (§3.1) — gros gain pour un effort moyen.
2. **Forcer le multi-tenancy** (Prisma middleware ou guard systématique) — sécurité.
3. **Tuer la dette CASL legacy** — moins de confusion.
4. **Repository pattern** sur les 5 modules cœur seulement.

C'est 80% de la valeur de l'hexagonal pour 20% de l'effort.

---

## 7. L'architecture actuelle — Bonne ou pas ?

### Verdict nuancé

**Les bons côtés** :
- ✅ Le module `platform/authz` est **bien fait** (hexagonal propre, ports/adapters).
- ✅ Le check `isRoot` est **bien centralisé** (pas de leak métier).
- ✅ JWT minimaliste, enrichissement à la demande dans le guard.
- ✅ Decisions explicites (`Decisions.allow()` / `Decisions.denyXxx()`).

**Les mauvais côtés** :
- ❌ Le scope est **fragmenté en 3 endroits** (vrai problème de cohérence).
- ❌ Multi-tenancy **non systématique** (risque de data leak).
- ❌ Modules métier en NestJS classique sans abstraction (tests difficiles).
- ❌ Deux systèmes RBAC en parallèle (legacy CASL + nouveau hexagonal).
- ❌ Pas d'audit (mentionné dans la doc v1, pas implémenté).

### Note globale

> **6/10**. L'autorisation **core** est bonne. Le **wiring métier** est faible. Ce n'est pas un mauvais code, mais il est en transition et les transitions inachevées sont les pires (deux mondes coexistent).

---

## 8. Plan pragmatique recommandé (par ordre de ROI décroissant)

| # | Action | Effort | Valeur |
|---|--------|--------|--------|
| 1 | Centraliser la résolution scope → un seul `ScopeResolver` qui sort un `PrismaWhereInput` | 2–3 j | 🔥🔥🔥 |
| 2 | Prisma middleware d'injection `org_id` (ou guard global) | 1–2 j | 🔥🔥🔥 |
| 3 | Supprimer ou rebrancher CASL legacy (`src/authorization/`) | 1 j | 🔥🔥 |
| 4 | Mettre en place le `UserSystemAccess` (doc v1) + virer `is_root`/`is_platform` du `Role` | 2 j | 🔥🔥 |
| 5 | Audit minimal (login / switch-org / actions root) | 2 j | 🔥🔥 |
| 6 | Repository pattern sur les 5 modules cœur (attendees, registrations, events, organizations, users) | 5–10 j | 🔥 |
| 7 | Hexagonal complet sur les autres modules | 20–30 j | 🟡 |

→ **Étapes 1 à 5 = vrai gain immédiat**. Le reste, seulement si l'équipe grandit ou que les tests deviennent prioritaires.

---

## 9. Réponses directes à tes questions

> **Le check `isRoot` est-il déjà centralisé aujourd'hui ?**

**Oui**. Aucun `if user.isRoot` dans les services métier. Il est dans 6 fichiers, tous dans `auth/` ou `platform/authz/`. Tu respectes déjà la règle d'or.

> **Comment les permissions sont gérées aujourd'hui ?**

Stack : `JwtAuthGuard` → `RequirePermissionGuard` → `AuthorizationService.can()` → check rôle/permission/scope. Le JWT ne porte pas les permissions, elles sont chargées à chaque requête depuis la DB par l'adapter.

> **Est-ce que je peux ajouter d'autres axes ?**

Oui (ex: tags, projets), mais **avant**, centralise le scope. Sinon chaque nouvel axe sera dupliqué à 3 endroits.

> **Est-ce facile de migrer vers hexagonal ?**

**Faisable, pas trivial.** ~10–12 modules à toucher. ~1–3 jours par module. Le `platform/authz` est déjà hexagonal et sert de modèle.

> **Combien de modules à migrer ?**

10 à 12 modules métier (sur 21). 5 sont vraiment critiques.

> **L'architecture actuelle est-elle bonne ?**

**Moyennement.** Le RBAC est propre. Le métier est NestJS basique. Le vrai problème n'est pas "hexagonal vs pas hexagonal", c'est la **fragmentation du scope** et le **multi-tenancy non systématique**. Règle ces deux points avant de penser hexagonal.
