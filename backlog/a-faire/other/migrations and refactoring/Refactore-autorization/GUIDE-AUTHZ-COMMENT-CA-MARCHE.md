# Guide pédagogique — Comment marche le système d'autorisation

> Ce document explique la mécanique complète du système d'autorisation, brique par brique :
> requête HTTP → guards → ports → adapters → décision. Lecture conseillée pour quiconque
> ajoute un nouveau controller, un nouveau guard, ou doit débugger un 403.

---

## 1. Le vocabulaire (à lire en premier)

Avant de plonger, ces 4 mots sont partout dans le code. Comprendre leur rôle évite 80% de la confusion.

### Controller
Une classe NestJS qui expose des routes HTTP (`@Get`, `@Post`, etc.). C'est la **porte d'entrée** d'une requête. Elle ne contient (presque) pas de logique métier — elle valide les inputs, appelle un service, renvoie une réponse.

```ts
@Controller('events')
export class EventsController {
  @Get(':id')
  findOne(@Param('id') id: string) { ... }
}
```

### Guard
Une classe NestJS qui répond à **une seule question** : « est-ce que cette requête a le droit de passer ? ». Elle s'exécute **avant** le controller. Si elle renvoie `false` ou throw, la requête est rejetée (401/403).

Exemples dans notre code :
- `JwtAuthGuard` : « est-ce que le token JWT est valide ? »
- `PermissionsGuard` : « est-ce que l'utilisateur a la permission requise ? »
- `TenantContextGuard` : « est-ce que l'utilisateur a une org active ? »

### Decorator
Une fonction qui **attache une métadonnée** à un controller ou une méthode. Cette métadonnée est ensuite **lue par un guard** via le `Reflector` de NestJS.

```ts
@Permissions('events.read:org')   // ← decorator qui dit "il faut cette permission"
@Get(':id')
findOne() { ... }
```

Le `PermissionsGuard` lit ensuite cette métadonnée et fait la vérification.

**Idée clé** : le decorator **ne fait rien tout seul**. Il écrit une étiquette. C'est le guard qui agit.

### Port (et adapter)
Vocabulaire d'**architecture hexagonale**. C'est juste deux mots compliqués pour une idée simple :

- **Port** = une **interface TypeScript**. Elle décrit ce qu'on veut faire, **sans** dire comment.
- **Adapter** = une **classe concrète** qui implémente le port. C'est elle qui fait vraiment le travail (souvent en parlant à la base de données ou à un service externe).

Pourquoi cette séparation ? Pour que la **logique métier** (le « core ») ne dépende **pas** d'une techno précise (Prisma, Postgres, etc.). On peut remplacer l'adapter sans toucher au reste.

#### Exemple concret dans le projet

Le port :
```ts
// src/platform/authz/ports/auth-context.port.ts
export interface AuthContextPort {
  buildAuthContext(jwtPayload: JwtPayload): Promise<AuthContext>;
}
```
→ Cette interface dit : « il existe quelque part un truc qui sait construire un `AuthContext` à partir d'un JWT. »

L'adapter :
```ts
// src/platform/authz/adapters/db/prisma-auth-context.adapter.ts
@Injectable()
export class PrismaAuthContextAdapter implements AuthContextPort {
  constructor(private prisma: PrismaService) {}

  async buildAuthContext(jwtPayload: JwtPayload): Promise<AuthContext> {
    const systemAccess = await this.prisma.userSystemAccess.findUnique(...);
    return { userId: jwtPayload.sub, isRoot: systemAccess?.is_root === true, ... };
  }
}
```
→ Cette classe **fait vraiment le travail** : elle parle à Prisma.

Les guards (`PermissionsGuard`, `RequirePermissionGuard`) ne connaissent que **le port**. Si demain on change Prisma pour Mongoose, on remplace juste l'adapter. Les guards ne bougent pas.

---

## 2. Le voyage d'une requête HTTP

Suivons une requête `GET /events/abc-123` faite par un utilisateur connecté.

```
┌─────────────────┐
│  HTTP request    │  GET /events/abc-123
│  Bearer xxx.jwt  │  Authorization: Bearer eyJhbGc...
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. JwtAuthGuard (src/common/guards/jwt-auth.guard.ts)          │
│  ─────────────────────────────────────────────                   │
│  - Lit l'en-tête Authorization                                   │
│  - Délègue à JwtStrategy.validate(payload)                       │
│  - JwtStrategy vérifie la signature, charge l'user en DB         │
│  - Renvoie le JwtPayload : { sub, currentOrgId, permissions }    │
│  - NestJS injecte ce payload dans req.user                       │
│                                                                  │
│  ⚙️ Bypass : si @Public() est posé sur la route, on saute       │
└────────┬────────────────────────────────────────────────────────┘
         │  req.user = { sub: 'u1', currentOrgId: 'org-9', ... }
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. PermissionsGuard (src/common/guards/permissions.guard.ts)   │
│  ─────────────────────────────────────────────                   │
│  - Lit la métadonnée @Permissions('events.read:org')             │
│  - Si pas de @Permissions → laisse passer                        │
│  - Sinon :                                                       │
│      a) Appelle authContextPort.buildAuthContext(req.user)       │
│         → l'adapter Prisma va chercher isRoot en DB              │
│         → renvoie { userId, isRoot, currentOrgId }               │
│      b) Enrichit req.user.isRoot (utile pour les controllers)    │
│      c) Pour chaque permission requise :                         │
│         authorizationService.can(perm, authContext, ctx)         │
│         → décision { allowed: true/false, code, details }        │
│      d) Si autorisé : enrichit req.user.permissions, return true │
│      e) Sinon : throw 403 ForbiddenException                     │
└────────┬────────────────────────────────────────────────────────┘
         │  ✅ Autorisé
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Controller (events.controller.ts)                            │
│  ─────────────────────────────────────────────                   │
│  @Permissions('events.read:org')                                 │
│  @Get(':id')                                                     │
│  findOne(@Param('id') id, @Req() req) {                          │
│    // req.user a maintenant : sub, currentOrgId, isRoot,         │
│    //                         permissions[]                       │
│    return this.eventsService.findOne(id, req.user.currentOrgId); │
│  }                                                                │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
   HTTP 200 + JSON
```

**À retenir** : les guards s'exécutent **dans l'ordre** où ils sont listés dans `@UseGuards(...)`. Si un guard throw, les suivants ne s'exécutent pas et le controller non plus.

---

## 3. Les fichiers du système (rôle de chacun)

### 3.1 Le JWT — la « carte d'identité » de la requête

| Fichier | Rôle |
|---------|------|
| `src/auth/interfaces/jwt-payload.interface.ts` | **Source de vérité** du shape du token. C'est ce qui est *vraiment* signé et envoyé au client. |
| `src/auth/jwt.strategy.ts` | Passport-JWT. Vérifie la signature + charge l'user en DB. Renvoie ce qui sera mis dans `req.user`. |
| `src/auth/auth.service.ts` | Construit le payload et le signe (login, refresh, switch-org). |

Shape actuel du JWT (Palier 3b, sans `mode`) :
```ts
{
  sub: string;              // userId
  currentOrgId?: string;    // org active
  permissions?: string[];   // permissions au format "code:scope"
  iat?: number;
  exp?: number;
}
```

### 3.2 Les guards — les « videurs »

| Fichier | Question posée |
|---------|----------------|
| `src/common/guards/jwt-auth.guard.ts` | Le token est-il valide ? Bypass si `@Public()`. |
| `src/common/guards/permissions.guard.ts` | L'utilisateur a-t-il la permission `@Permissions(...)` ? Pratique le bypass root via `AuthContext.isRoot`. |
| `src/common/guards/tenant-context.guard.ts` | L'utilisateur a-t-il un `currentOrgId` ? Bypass si `@SkipTenantContext()` ou `@Public()`. |
| `src/platform/authz/adapters/http/guards/require-permission.guard.ts` | Variante « adapter » de `PermissionsGuard`, utilisée par les controllers de la couche `platform/authz`. Même logique. |

### 3.3 Les decorators — les « étiquettes »

| Fichier | Pose une étiquette qui dit… |
|---------|------------------------------|
| `src/common/decorators/permissions.decorator.ts` (`@Permissions(...)`) | « cette route exige ces permissions » |
| `src/common/guards/jwt-auth.guard.ts` (`@Public()`) | « cette route est accessible sans token » |
| `src/common/guards/tenant-context.guard.ts` (`@SkipTenantContext()`) | « pas besoin d'org active pour cette route » |
| `src/common/decorators/current-user.decorator.ts` (`@CurrentUser()`) | injecte `req.user` directement comme paramètre |

### 3.4 Les ports — les « contrats »

| Fichier | Décrit l'interface de… |
|---------|------------------------|
| `src/platform/authz/ports/auth-context.port.ts` | … construire un `AuthContext` à partir du JWT. Un seul port, deux interfaces dedans : `JwtPayload` et `AuthContextPort`. |
| `src/platform/authz/ports/rbac-query.port.ts` | … interroger le RBAC (rôles, permissions) en base. |

### 3.5 Les adapters — les « implémentations concrètes »

| Fichier | Implémente quoi |
|---------|-----------------|
| `src/platform/authz/adapters/db/prisma-auth-context.adapter.ts` | `AuthContextPort` avec Prisma. C'est lui qui lit `UserSystemAccess` pour `isRoot`. |
| `src/platform/authz/adapters/db/prisma-rbac-query.adapter.ts` | `RbacQueryPort` avec Prisma. Lit les rôles/permissions de l'user. |
| `src/platform/authz/adapters/http/guards/require-permission.guard.ts` | Adapter HTTP qui transforme « route HTTP » en « décision RBAC ». |

### 3.6 Le core — la « logique métier » de l'authz

| Fichier | Rôle |
|---------|------|
| `src/platform/authz/core/authorization.service.ts` | Le cerveau. `can(permission, authContext, ctx)` → décision. |
| `src/platform/authz/core/permission-resolver.ts` | Résout la liste effective des permissions d'un user (rôles → grants). |
| `src/platform/authz/core/types.ts` | Types : `AuthContext`, `Decision`, `Grant`, etc. |

**Idée importante** : le core **ne connaît ni HTTP, ni Prisma, ni NestJS**. Il prend des objets en entrée, renvoie une décision. C'est ce qui le rend testable unitairement.

---

## 4. Comprendre le bypass `isRoot` (cas C2 du review)

C'est la partie qui t'a posé problème. Reprenons en détail.

### Le besoin métier
Un **user root** doit pouvoir générer un badge **pour n'importe quelle org**, sans être membre de cette org. C'est un bypass exceptionnel pour le support / l'admin.

### Comment c'est exprimé en code

Dans `badge-generation.controller.ts`, le service a besoin d'un `orgId` :
- soit l'org de l'user → la requête est limitée à cette org
- soit `null` → le service ne filtre pas et accède à n'importe quelle org

```ts
const allowAny = req.user.isRoot === true && req.user.permissions?.some(p => p.endsWith(':any'));
const orgId = allowAny ? null : req.user.currentOrgId;
```

Décomposition :

1. **`req.user.isRoot === true`** : l'utilisateur est marqué root dans `UserSystemAccess`. Ce flag est posé par `PermissionsGuard` après avoir construit l'`AuthContext`. C'est la **seule** condition pour avoir le droit de bypasser.

2. **`req.user.permissions?.some(p => p.endsWith(':any'))`** : ceinture **et** bretelles. On vérifie qu'il a *aussi* au moins une permission de scope `:any` dans le contexte courant. C'est une double sécurité : un root sans aucune permission `:any` ne déclenche pas le bypass.

3. **`orgId = allowAny ? null : req.user.currentOrgId`** : si bypass autorisé, on passe `null` au service → pas de filtre. Sinon, on passe l'org active de l'user → filtre normal.

### Pourquoi le code initial était cassé (bug C2)

Le code pré-fix était :
```ts
const allowAny = req.user.mode === 'platform' && req.user.permissions?.some(...);
```

Mais `req.user.mode` **n'existe plus** (retiré au Palier 3b). Donc `allowAny` valait **toujours `false`**. Conséquence :
- un root qui voulait générer un badge dans une autre org → `orgId = req.user.currentOrgId` (son org à lui, pas l'autre)
- le service filtrait sur cette org → la registration cible n'était pas trouvée → 404

C'était silencieux : pas d'erreur 403, juste « ressource introuvable ». Difficile à débugger sans regarder le code.

### Le fix
Remplacer `req.user.mode === 'platform'` par `req.user.isRoot === true`. Mais pour que ce flag existe sur `req.user`, il faut que **quelqu'un l'y ait mis**. C'est pour ça qu'on a aussi modifié `PermissionsGuard` et `RequirePermissionGuard` pour qu'ils enrichissent `req.user.isRoot` après avoir construit l'`AuthContext`.

```
PermissionsGuard.canActivate()
  ↓
authContextPort.buildAuthContext(jwt)   ← lit UserSystemAccess en DB
  ↓
authContext = { userId, isRoot: true, currentOrgId }
  ↓
(request.user as any).isRoot = authContext.isRoot   ← enrichissement
  ↓
le controller peut maintenant lire req.user.isRoot
```

### Pourquoi cette double-vérification (`isRoot` + `:any`) ?

C'est une protection contre une éventuelle régression de configuration :
- Si un jour quelqu'un fait `isRoot = true` sur un user qui ne devrait pas l'être (mauvaise migration, accident), il aura quand même besoin de `:any` pour passer.
- Le scope `:any` est lui-même rare et explicite dans les rôles.
- Les deux ensemble = très peu de risque de bypass accidentel.

C'est ce qu'on appelle **defense in depth** : plusieurs lignes de défense indépendantes.

---

## 5. Comment ajouter un nouveau controller protégé (recette)

```ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { PermissionsGuard } from '../../common/guards/permissions.guard';
import { Permissions } from '../../common/decorators/permissions.decorator';

@Controller('my-resource')
@UseGuards(JwtAuthGuard, PermissionsGuard)   // ← ordre important
export class MyResourceController {

  @Get()
  @Permissions('myresource.read:org')
  findAll(@Req() req) {
    // req.user.currentOrgId est garanti présent
    // req.user.isRoot est disponible si tu as un cas de bypass
    return this.service.findAll(req.user.currentOrgId);
  }
}
```

**Checklist mentale** :
1. `JwtAuthGuard` toujours présent si la route est privée.
2. `PermissionsGuard` après `JwtAuthGuard`.
3. Une seule permission par méthode (en général). Format `domaine.action:scope`.
4. Si tu fais un bypass root, utilise `req.user.isRoot === true`. Jamais `req.user.mode`.
5. Si tu lis `req.user.currentOrgId`, c'est déjà garanti par `PermissionsGuard` (il throw sinon).

---

## 6. Comment débugger un 403

1. **Active les logs `debug` sur `PermissionsGuard`** : il logge `check permission=... allowed=... code=...`.
2. **Regarde le `code`** du `decision` :
   - `NO_PERMISSION` : l'user n'a aucun grant pour cette permission
   - `WRONG_SCOPE` : il a la permission mais sur un scope incompatible
   - `WRONG_ORG` : la ressource n'est pas dans son org
3. **Vérifie le contenu de `req.user`** : log `req.user` dans le controller. Si `currentOrgId` manque, c'est un problème de switch-org.
4. **Vérifie `UserSystemAccess`** en base si tu attendais un bypass root.
5. **Vérifie les rôles** : `Role` + `RolePermission` + `UserRole` doivent être cohérents.

---

## 7. Erreurs courantes (à ne pas refaire)

| Erreur | Pourquoi c'est mal | Bonne pratique |
|--------|--------------------|----------------|
| Lire `req.user.mode` | Ce champ n'existe plus depuis Palier 3b | Lire `req.user.isRoot` |
| Oublier `PermissionsGuard` dans `@UseGuards` | Le `@Permissions(...)` ne sera pas vérifié → route ouverte | Toujours le mettre |
| Mettre `@UseGuards(PermissionsGuard, JwtAuthGuard)` (ordre inversé) | `PermissionsGuard` lira un `req.user` vide | Toujours `JwtAuthGuard` en premier |
| Faire un check de permission « à la main » dans le service | Logique éparpillée, intestable | Utiliser le decorator + guard |
| Ajouter une condition métier dans un guard | Les guards doivent rester sur l'autorisation pure | Mettre la logique métier dans le service |
| Modifier le port sans mettre à jour l'adapter | Erreur de compilation TS, bug runtime | Toujours regarder qui implémente le port |

---

## 8. Pour aller plus loin

- `docs/Refactore-autorization/Refactore authorization v1 = tenant-first only + root override.md` : la spec d'origine du refactor.
- `docs/Refactore-autorization/Comprendre-passport-jwt-strategy.md` : comment Passport-JWT se branche dans NestJS.
- `docs/Refactore-autorization/Phase 2 — Comptes internes non-root.md` : la suite (comptes internes + APP_GUARD global).
- `docs/Refactore-autorization/REVIEW-AUTHZ-V1.md` : revue de code complète de la V1.

---

## TL;DR

```
Decorator   = étiquette posée sur la route (ne fait rien tout seul)
Guard       = lit l'étiquette et décide si la requête passe
Controller  = orchestre la requête (mais ne décide jamais des droits)
Port        = interface (le contrat)
Adapter     = implémentation concrète (parle à Prisma / HTTP / etc.)
Core        = logique pure (ne connaît ni HTTP, ni Prisma)

req.user contient :
  - sub, currentOrgId, permissions   (mis par JwtStrategy)
  - isRoot                           (enrichi par PermissionsGuard)

Pour bypasser un check métier en tant que root : req.user.isRoot === true
JAMAIS : req.user.mode (le champ n'existe plus)
```
