# Palier 3c — Cleanup back post-refactor

> **Statut : ✅ TERMINÉ** — commit `c6d8e7a` sur branche `Refactore-authorization-v1`
> **Date : 24 avril 2026**

## Contexte

Après les Paliers 1 (UserSystemAccess), 2 (refactor du code), 3a (drop platform) et 3b (rename TenantUserRole → UserRole), il restait quelques petites traces de l'ancien système et un bug fonctionnel à corriger avant de pouvoir attaquer sereinement le front.

Ce palier ne touche **ni** au schéma DB, **ni** aux contrats HTTP. C'est uniquement de l'hygiène code + un fix de bug.

## Items traités

### 1. Fix `/auth/policy` pour root sans rôle dans l'org

**Fichier :** `src/auth/auth.service.ts::getUserAbility`

**Symptôme :** un user root (UserSystemAccess.is_root = true) qui switche sur une organisation où il n'a pas de UserRole recevait une 404 `User has no role in this organization`. Tous les autres endpoints répondaient correctement (le bypass root est appliqué dans `AuthorizationService.can`), mais `/auth/policy` jetait une exception.

**Cause :** la méthode `getUserAbility` cherche un `UserRole` et throw `NotFoundException` si rien trouvé, sans vérifier le statut root.

**Fix :** avant de throw, on vérifie `UserSystemAccess.is_root`. Si true, on retourne un payload synthétique `{ orgId, modules, grants: [] }`. Les guards continuent d'autoriser via le bypass — l'endpoint répond simplement 200 pour que le front puisse rendre l'UI sans erreur.

**Validation :**

```bash
# Avant : 404 {"detail":"User has no role in this organization"}
# Après : 200 {"rules":[],"version":"1.0.0"}
curl -H "Authorization: Bearer $TOKEN_ACME_ROOT" http://localhost:3000/api/auth/policy
```

### 2. `console.log` debug RBAC → `Logger.debug`

**Fichiers :**

- `src/common/guards/permissions.guard.ts`
- `src/platform/authz/adapters/http/guards/require-permission.guard.ts`
- `src/platform/authz/core/authorization.service.ts`

**Avant :** 3 `console.log` bruyants émis à chaque requête authentifiée. Pollue les logs Cloud Run et empêche de filtrer proprement.

**Après :** `Logger` Nest scoped (`new Logger(ClassName.name)`) avec `logger.debug()`. Silencieux par défaut, visible uniquement avec `LOG_LEVEL=debug` ou config Nest équivalente.

### 3. Cleanup commentaire mort `roles.service.ts`

**Fichier :** `src/modules/roles/roles.service.ts::findUserRole`

L'ancien `// TODO: Cette méthode utilise l'ancien modèle single-tenant` est remplacé par un commentaire clair indiquant que la méthode est un vestige (elle throw déjà), et qu'elle sera supprimée définitivement (avec son endpoint dans `roles.controller.ts`) dans une PR ultérieure une fois confirmé qu'aucun client ne l'appelle plus.

### 4. (Vérification) `seed-role-users.sql`

`grep` confirme que ce fichier (qui contient encore le nom `tenant_user_roles` dans des INSERT) n'est référencé nulle part dans le code applicatif. Il est déjà rangé dans `scripts/legacy/sql/`. Pas d'action nécessaire — il sera traité avec le prochain ménage des seeds historiques.

## Validation finale

| Vérification | Résultat |
|---|---|
| `npx tsc --noEmit` | ✅ 0 erreur |
| `bash scripts/test-palier2-curl.sh` | ✅ tous tests verts |
| `/auth/policy` sur Acme (root sans rôle) | ✅ 200 (était 404) |
| Login + me/orgs + switch-org | ✅ inchangé |
| Bypass root sur `/organizations` | ✅ inchangé |
| Sans token | ✅ 401 |

## Ce que NE fait PAS ce palier

- ❌ Touche aux contrats HTTP (les champs `mode` et `isPlatform` restent gelés à `'tenant'` / `false` en réponse — c'est le boulot du **Palier 4**)
- ❌ Touche au schéma DB
- ❌ Migration de données

## Prochaine étape

→ **Palier Front + Mobile** (PRs séparées, branches dédiées, voir `02-REFACTOR-FRONT.md` et `03-REFACTOR-MOBILE.md`)
