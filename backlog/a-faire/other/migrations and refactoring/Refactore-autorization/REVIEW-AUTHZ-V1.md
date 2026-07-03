# Code Review — Refactoring `Refactore-authorization-v1`

> **Date :** 4 mai 2026  
> **Reviewer :** GitHub Copilot (senior review automatisée)  
> **Branches :** `Refactore-authorization-v1` (back + front)  
> **tsc back :** ✅ vert | **tsc front :** ✅ vert

---

## Résumé du refactoring compris

Migration vers un modèle d'autorisation **tenant-first only + root override** :

- Plus de mode `platform` runtime — tout user (root inclus) agit dans une org
- `Role` purifié : suppression de `is_root` / `is_platform`
- Création de `UserSystemAccess` (table dédiée pour le flag `is_root`)
- Renommage `TenantUserRole` → `UserRole`
- Endpoints clés : `POST /auth/login`, `GET /auth/me/orgs`, `POST /auth/switch-org`, `GET /auth/policy`
- JWT minimal : `{ sub, currentOrgId?, permissions?, isRoot? }` — le champ `mode` est supprimé
- Front aligné : `lastOrgId` persisté, `SelectOrganization`, `OrganizationSwitcher`, reset cache + reload policy au switch

---

## Changements détectés dans les commits

### Back (`Refactore-authorization-v1`) — ~10 commits significatifs

| Commit | Description |
|--------|-------------|
| `1db795a` | Palier 1 — démarrage refactor + documentation |
| `490a2dd` | Palier 2 — rewiring code (geler la platform) |
| `fbb8e56` | Palier 3a — grand nettoyage schéma Prisma |
| `539c198` | Endpoints `switch-org` + `available-orgs` |
| `59e2bdd` | Palier 3b — rename `TenantUserRole` → `UserRole`, drop `mode`/`isPlatform` types internes |
| `c6d8e7a` | Palier 3c — fix `/auth/policy` bypass root + Logger |
| `a153c4e` → `1167f2a` | Plans détaillés (front, mobile, tests, déploiement, UX) |

### Front (`Refactore-authorization-v1`) — 2 commits + working copy non commité

| Commit | Description |
|--------|-------------|
| `104846e` | F1+F2+F3 — endpoints `me/orgs` & `switch-org`, retrait usage `isPlatform` |
| `9c1e6df` | F-UX étape 1 — `lastOrgId`, hooks `useMyOrgs` / `useIsMultiOrg` |
| _(non commité)_ | `SelectOrganization/`, `OrganizationSwitcher`, `Header`, `AuthLayout`, `authApi` |

### Mobile

❌ **Aucun commit lié à l'authz v1.** Non aligné.

---

## Points positifs

- ✅ Schéma Prisma propre : `Role` purgé de `is_root`/`is_platform`, `UserSystemAccess` créé, `UserRole` bien renommé
- ✅ Login flow propre : détection root via `UserSystemAccess.is_root` ; JWT initial sans org pour multi-org → forçage du choix
- ✅ `verifyOrgAccess()` correct : root accède à toutes les orgs, tenant restreint à `OrgUser`
- ✅ `AuthorizationService.can()` : bypass root immédiat, indépendant des grants
- ✅ `getMyOrgs` : root reçoit toutes les orgs avec `role:'ROOT'`, tenant uniquement ses memberships
- ✅ Data scoping manuel mais explicite (~30 occurrences `org_id` vérifiées dans les services)
- ✅ Front : `isPlatform` retiré de toute la logique métier (seul typage HTTP back-compat conservé, marqué `@deprecated`)
- ✅ `switchOrg` mutation : `resetApiState()` + `setRules([])` + `invalidatesTags: ['Auth','Policy']` — propre
- ✅ `redux-persist` pour `lastOrgId`
- ✅ Endpoints back/front cohérents (URL, payloads, tags RTK Query)
- ✅ `tsc --noEmit` vert sur back ET front

---

## Problèmes / risques détectés

### 🔴 Critiques — bloquants pour merge

#### C1 — `TenantContextGuard` cassé et inutilisé

**Fichier :** `src/common/guards/tenant-context.guard.ts` ligne 18

```ts
// ACTUEL — CASSÉ
if (user.mode !== 'tenant' || !user.currentOrgId) {
  throw new BadRequestException('No organization context...');
}
```

Le JWT réel n'a **plus** de champ `mode` depuis le Palier 3b. Ce guard rejetterait toutes les requêtes — sauf qu'il n'est **appliqué nulle part**. Le filet de sécurité censé matérialiser le principe central du refactor ("root DOIT avoir un `activeOrgId`") **n'existe donc pas**.

**Fix :**
```ts
// CORRIGÉ
if (!user.currentOrgId) {
  throw new BadRequestException('No organization context. Please switch to an organization first.');
}
```
Puis enregistrer en `APP_GUARD` global avec un décorateur `@SkipTenantContext()` sur les routes d'exception (login, refresh, me/orgs, switch-org, public).

---

#### C2 — `badge-generation.controller.ts` : condition morte

**Fichier :** `src/modules/badge-generation/badge-generation.controller.ts` ligne 270

```ts
// ACTUEL — CASSÉ
const allowAny = req.user.mode === 'platform' && req.user.permissions?.some((p: string) =>
  p.endsWith(':any')
);
```

`req.user.mode` est toujours `undefined` → `allowAny` est toujours `false` → **un root perd son accès cross-org au badge preview**.

**Fix :**
```ts
// CORRIGÉ
const allowAny = req.user.isRoot === true;
```
Ou déléguer à `AuthorizationService.can()` avec une permission `:any`.

---

#### C3 — Interface `JwtPayload` divergente du JWT réel

**Fichier :** `src/platform/authz/ports/auth-context.port.ts` ligne 16

```ts
// ACTUEL — INCOHÉRENT
export interface JwtPayload {
  sub: string;
  mode: 'tenant' | 'platform';   // ← champ inexistant dans le JWT réel
  currentOrgId?: string;
  iat: number;
  exp: number;
}
```

Le JWT réel (`jwt-payload.interface.ts`) n'émet pas `mode`. Toute consommation TypeScript de ce port croit que `mode` existe → bugs silencieux (cf. C1 et C2 en découlent directement).

**Fix :** aligner sur la vraie structure :
```ts
export interface JwtPayload {
  sub: string;
  id?: string;
  currentOrgId?: string;
  permissions?: string[];
  iat: number;
  exp: number;
}
```

---

### 🟡 Majeurs — à régler avant prod

#### M1 — `AuditService` totalement absent

Le plan marque explicitement "Phase 1 minimum requis" : login / switch-org / actions root. Aucune trace d'un service d'audit dans le code. Pour les actions root, c'est une **exigence compliance** du refactor.

#### M2 — Aucun guard tenant systématique sur plusieurs controllers

Plusieurs controllers actifs n'utilisent que `JwtAuthGuard` sans vérification de `currentOrgId` ni de permissions :
- `src/modules/event-tables/event-tables.controller.ts`
- `src/modules/sessions/sessions.controller.ts`

À auditer : si ces routes manipulent des données tenant → ajouter `RequirePermissionGuard` (passe par `AuthContext` → `currentOrgId`) ou activer le fix C1 en global.

#### M3 — `getPolicyRules` ne revalide pas l'accès à `currentOrgId`

`auth.service.ts` rejette si pas d'org, mais ne rappelle pas `verifyOrgAccess(userId, currentOrgId)`. Un JWT réutilisé après révocation pourrait charger les rules d'une org où l'user n'est plus membre. Faible probabilité (signature JWT), mais facile à durcir.

#### M4 — Mobile non touché

L'app mobile (`attendee-ems-mobile`) n'a aucun commit sur ce refactor. Si elle parle au même back en prod, vérifier qu'elle ne lit pas `mode` / `isPlatform`. À traiter **avant** le Palier 4 (suppression définitive des champs HTTP dépréciés).

#### M5 — Drift detection front incomplète (F-UX.7)

| Comportement | État |
|---|---|
| `refetchOnFocus: true` | ✅ |
| Toast "vous avez maintenant accès à X" | ❌ |
| Forced re-switch si org active disparaît | ❌ |
| Force logout si liste vide | ❌ |

Pas bloquant pour merge, à tracker en PR follow-up.

#### M6 — Composants front non commités

`SelectOrganization/`, `OrganizationSwitcher`, modifications `Header`, `AuthLayout`, `Login`, `authApi` sont dans le working copy et non versionnés. À commiter avant merge.

---

### 🟢 Mineurs

- `AuthContext.isPlatform` encore dans `src/platform/authz/core/types.ts` — à supprimer au Palier 4
- `console.log` résiduels dans `Login/index.tsx` et `badge-generation.controller.ts` — à passer en `Logger`
- Commentaire JSDoc de `current-user.decorator.ts` mentionne `user.mode` — à corriger
- Pas de selector `selectIsRoot` centralisé côté front (détection via `user.roles.includes('ROOT')` éparpillée)
- `isPlatform: false` figé dans `getAvailableOrgs` — accepté car `@deprecated`, à inscrire au Palier 4

---

## Modules impactés à vérifier

### Back

| Fichier | Priorité | Action |
|---------|----------|--------|
| `src/common/guards/tenant-context.guard.ts` | 🔴 C1 | Fix `mode` → activer global |
| `src/platform/authz/ports/auth-context.port.ts` | 🔴 C3 | Aligner `JwtPayload` |
| `src/modules/badge-generation/badge-generation.controller.ts` | 🔴 C2 | Remplacer `mode` par `isRoot` |
| `src/platform/authz/core/types.ts` | 🟡 M1 | Retirer `isPlatform` au Palier 4 |
| `src/auth/auth.service.ts` (`getPolicyRules`) | 🟡 M3 | Revalider accès org |
| `event-tables.controller.ts`, `sessions.controller.ts` | 🟡 M2 | Audit guards |
| _(à créer)_ `AuditService` | 🟡 M1 | login + switch-org + actions root |
| `src/auth/interfaces/jwt-payload.interface.ts` | 🟢 | Référence de vérité pour le type |

### Front

| Fichier | Priorité | Action |
|---------|----------|--------|
| `src/pages/SelectOrganization/index.tsx` | 🟡 M6 | Commiter |
| `src/features/auth/components/OrganizationSwitcher.tsx` | 🟡 M6 | Commiter |
| `src/widgets/Header/index.tsx` | 🟡 M6 | Commiter |
| `src/widgets/layouts/AuthLayout.tsx` | 🟡 M6 | Commiter |
| `src/features/auth/api/authApi.ts` | 🟢 | Drop `isPlatform` au Palier 4 |
| `src/features/auth/model/sessionSlice.ts` | 🟢 | Ajouter `selectIsRoot` |

### Mobile

| Scope | Priorité | Action |
|-------|----------|--------|
| Tout le module auth | 🟡 M4 | Audit avant Palier 4 back |

---

## Tests exécutés ou à exécuter

### Déjà exécutés

| Test | Résultat |
|------|----------|
| `tsc --noEmit` (back) | ✅ vert |
| `tsc --noEmit` (front) | ✅ vert |

### À exécuter avant merge

```bash
# Back
npm run lint
npm test

# Front
npm run lint
npx tsc --noEmit
npm test
npx playwright test  # si configuré
```

### Smoke tests curl

```bash
# 1. Login root
POST /auth/login { email: root@..., password }
→ access_token sans currentOrgId

# 2. Login tenant mono-org
POST /auth/login { email, password }
→ access_token (selon config back, peut contenir orgId)

# 3. Me/orgs root → doit retourner TOUTES les orgs avec role:'ROOT'
GET /auth/me/orgs (token root sans org)

# 4. Me/orgs tenant → seulement ses orgs
GET /auth/me/orgs (token tenant)

# 5. Switch-org root vers org X
POST /auth/switch-org { orgId: "uuid-org-X" }
→ nouveau token avec currentOrgId = "uuid-org-X"

# 6. Switch-org tenant vers org tierce
POST /auth/switch-org { orgId: "uuid-org-pas-la-sienne" }
→ 403 Forbidden

# 7. Policy avec org → rules correctes
GET /auth/policy (token avec currentOrgId)

# 8. Policy sans org → erreur
GET /auth/policy (token sans currentOrgId)

# 9. Badge preview root cross-org (après fix C2)
GET /events/:id/badges/:registrationId/preview (token root, org ≠ event.org_id)
→ doit fonctionner

# 10. Route tenant sans currentOrgId (après fix C1)
GET /events (token valide mais sans currentOrgId)
→ doit être bloqué par TenantContextGuard
```

---

## Checklist avant merge

- [ ] **C1** — Fix `TenantContextGuard` : retirer vérif `mode`, activer en global avec `@SkipTenantContext()`
- [ ] **C2** — Fix `badge-generation.controller.ts` : `mode === 'platform'` → `isRoot === true`
- [ ] **C3** — Aligner `JwtPayload` dans `auth-context.port.ts`
- [ ] Audit guards : `event-tables`, `sessions`, autres controllers sans permission guard
- [ ] `AuditService` minimal : login + switch-org + actions root
- [ ] Commiter les composants front : `SelectOrganization`, `OrganizationSwitcher`, `Header`, `AuthLayout`
- [ ] `npm run lint` vert back + front
- [ ] Tests unitaires verts back + front
- [ ] Smoke tests des 10 cas ci-dessus
- [ ] Vérifier mobile : impact prod ou non ?
- [ ] Documenter la PR follow-up : F-UX.7 drift detection + Palier 4

---

## Note générale

**7 / 10**

Le **design** est très solide : séparation root / rôle, schéma propre, bypass root au bon endroit (dans `AuthorizationService.can()`, pas dans les guards), frontière claire permissions vs contexte org. Le front s'aligne proprement avec cache reset total et reload policy au switch.

Le **-3** vient de l'**exécution incomplète sur la sécurité** :
- Le guard censé matérialiser le principe central du refactor est cassé ET non utilisé
- Un controller tient une condition morte basée sur `mode === 'platform'`
- L'audit (exigence explicite du plan) est totalement absent

Ces trois points contredisent directement les principes annoncés.

---

## Recommandation finale

> 🟡 **Merge avec réserves — bloquer tant que C1, C2, C3 ne sont pas fixés.**

Les trois fix sont mécaniques (< 1h). Recommandé de les faire dans la même PR avant merge.

M1 (audit), M2 (guards systématiques) et le commit des composants front peuvent partir en commit follow-up sur la même branche ou en PR trackée, **mais doivent être résolus avant tout déploiement en prod et avant le Palier 4** (suppression définitive des champs HTTP back-compat).

Le mobile doit faire l'objet d'une PR dédiée **avant** que les champs `mode`/`isPlatform` soient retirés de l'API.
