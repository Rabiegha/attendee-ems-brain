# Authorization v1 — Tenant-first + Root override

> **Principe** : tout est tenant. Le root est un accès système exceptionnel, pas un rôle.

---

## 1. Modèle v1 — Règles fondamentales

| Règle | Explication |
|-------|-------------|
| Tous les users métier sont des **users tenant** | Pas de user "platform-only" dans le runtime |
| Toute action métier se fait **dans une org** | Même le root doit avoir un `activeOrgId` |
| Un user peut appartenir à **plusieurs orgs** | Via la table `OrgUser` |
| Un user a **un rôle tenant par org** | Via `TenantUserRole` (1 rôle = 1 user + 1 org) |
| Les permissions viennent du **rôle** | Via `RolePermission` → `Permission(code, scope)` |
| Le scope `assigned` s'appuie sur **`event_access`** | Liaison user ↔ event pour limiter le périmètre |
| Certains comptes ont un **accès root système** | Séparé dans `UserSystemAccess`, hors du modèle `Role` |

---

## 2. Geler la platform

Le concept "platform" est **gelé** pour la v1 :

- ❌ Plus de JWT avec `mode: platform`
- ❌ `PlatformUserRole` et `PlatformUserOrgAccess` ne sont plus utilisées dans le runtime
- ✅ Les tables restent en base (pas de migration destructive) mais sont considérées **dormantes**
- ✅ Documenté : *"Authorization v1 = tenant-first only + root override"*

---

## 3. Root = accès système séparé

### Pourquoi sortir le root de `Role`

Le `is_root` dans `Role` est problématique :
- C'est un booléen qui "flotte" dans un objet fourre-tout
- Le root n'est pas un rôle — c'est une **capacité système exceptionnelle**
- Il ne vit pas dans le même monde que les rôles tenant

### Nouveau modèle

```prisma
model UserSystemAccess {
  user_id      String   @id @db.Uuid
  is_root      Boolean  @default(false)
  granted_by   String?  @db.Uuid       // Qui a donné l'accès
  granted_at   DateTime @default(now())
  reason       String?                 // Justification auditée
  user         User     @relation(fields: [user_id], references: [id], onDelete: Cascade)

  @@map("user_system_access")
}
```

### Comportement du root

- Le root **bypass toutes les permissions** dans `AuthorizationService.can()`
- Le root **DOIT** avoir un `activeOrgId` — toute action est traçable à une org
- Le root **n'a pas** de rôle tenant — il agit "en tant que root dans l'org X"
- Toutes les actions root sont **auditées** (qui, quoi, dans quelle org, quand)

### Root = bypass, mais PAS sans contexte

Le root bypass les **permissions**, pas le **contexte**. C'est une règle non-négociable.

```
❌ INTERDIT : root fait une action sans org
   → req.user = { sub: "root-id", isRoot: true, currentOrgId: null }
   → Le TenantContextGuard BLOQUE, même pour root

✅ OBLIGATOIRE : root agit toujours dans une org
   → req.user = { sub: "root-id", isRoot: true, currentOrgId: "org-xxx" }
   → Le guard laisse passer
   → Les queries filtrent par org_id
   → L'action est auditée avec l'org
```

**Pourquoi ?**

| Sans contexte org | Avec contexte org |
|-------------------|-------------------|
| `SELECT * FROM attendees` → TOUTE la base | `SELECT * FROM attendees WHERE org_id = ?` → données de l'org uniquement |
| Pas de traçabilité : "root a fait quoi, où ?" | Audit clair : "root a modifié X dans org Y" |
| Erreur silencieuse : le service reçoit `null` comme `orgId` | Comportement déterministe |
| Risque de data leak cross-tenant | Isolation tenant garantie |

**Conséquence sur le flow** :

```
Root se connecte
  → Login détecte UserSystemAccess.is_root = true
  → JWT émis SANS currentOrgId
  → Le front reçoit la liste de TOUTES les orgs (root voit tout)
  → Root DOIT appeler POST /auth/switch-org { orgId } avant toute action métier
  → Nouveau JWT avec currentOrgId
  → Maintenant il peut agir (avec bypass permissions)
```

**Le root ignore complètement `OrgUser` / `TenantUserRole`.**
Ses memberships ne sont **pas regardées** :
- ni pour décider de l'accès (l'accès est total via `isRoot`),
- ni pour l'UX (puisqu'il voit toutes les orgs de toute façon).

Il n'y a donc **pas** de branche "0 / 1 / N orgs" pour le root : c'est toujours "JWT sans org → choisir une org → switch-org".

**Différence avec un tenant normal** : le root peut `switch-org` vers **n'importe quelle org**. Le `verifyOrgAccess()` check `isRoot → true → accès à toutes les orgs`. Un tenant, lui, est restreint à ses orgs via `OrgUser`.

---

## 4. Nettoyer le modèle `Role`

Le `Role` doit devenir une **définition pure** :

```prisma
model Role {
  id              String   @id @default(uuid()) @db.Uuid
  code            String                         // ORG_ADMIN, ORG_STAFF, etc.
  name            String
  description     String?
  org_id          String?  @db.Uuid              // NULL = template système, non-NULL = rôle custom org
  level           Int      @default(99)          // Hiérarchie (1 = plus haut)
  is_system_role  Boolean  @default(false)       // Template non-modifiable

  rolePermissions RolePermission[]
  organization    Organization? @relation(fields: [org_id], references: [id])

  @@unique([org_id, code])
  @@map("roles")
}
```

**Supprimé** : `is_root`, `is_platform` — ces flags n'ont plus de raison d'être dans le modèle Role.

---

## 5. Autorisation = 100% serveur, 0% ambigu

### Principe

Le front peut afficher/masquer des éléments, mais **il n'est jamais la source de vérité**.

### Architecture

```
┌─────────────────────────────────────────┐
│         AuthorizationService            │
│  (unique source de vérité)              │
│                                         │
│  Entrée : user + activeOrgId            │
│  Sortie :                               │
│    • permissions effectives             │
│    • contraintes de scope               │
│    • filtres de périmètre (event_access) │
└────────────┬────────────────────────────┘
             │
     ┌───────┴───────┐
     ▼               ▼
  Guards           GET /auth/policy
  (enforcement     (snapshot pour le front CASL)
   côté serveur)
```

- **Tous** les endpoints passent par la même logique — pas "presque tous"
- Le front CASL est dérivé d'un **snapshot serveur** (`GET /auth/policy`), pas d'une logique reconstituée côté client

---

## 6. Forcer le tenant scope dans la data layer

**L'objectif** : oublier `org_id` doit être **difficile**, pas facile.

### Stratégie (par ordre de préférence)

| Option | Mécanisme | Avantage |
|--------|-----------|----------|
| A | Prisma middleware qui injecte automatiquement `org_id` | Transparent pour le dev |
| B | Repository layer obligatoire avec `org_id` en paramètre requis | Explicite + testable |
| C | Helper de query builder `withOrgScope(orgId, query)` | Léger à implémenter |
| D | Guard qui vérifie que `currentOrgId` n'est pas null avant le handler | Filet de sécurité |

> **Phase 1** : option D (guard) + option B/C dans les services existants.
> **Phase 2** : option A (Prisma middleware) pour l'enforcement systématique.

---

## 7. TenantUserRole — limitation consciente

### État actuel

1 rôle par user par org → c'est une **limitation produit assumée**, pas un axiome technique.

### Évolution préparée (pas implémentée)

Si besoin futur de rôles multiples ou permissions additionnelles :

```
Option A : TenantUserRoleAssignment (N lignes par user/org)
Option B : Rôle principal + permissions additionnelles par user/org
Option C : Grants multiples indépendants des rôles
```

> On documente ces options maintenant. On ne les implémente que si le besoin se confirme.

---

## 8. Audit — les actions à tracer

### Phase 1 (minimum requis)

| Action | Données tracées |
|--------|----------------|
| Login | userId, email, IP, userAgent, succès/échec |
| Switch org | userId, fromOrgId, toOrgId |
| Actions root | userId, orgId, action, ressource |

### Phase 2 (complet)

| Action | Données tracées |
|--------|----------------|
| Grant/revoke role | userId, targetUserId, orgId, roleCode |
| Accès support à une org | userId, orgId, raison |
| Export de données | userId, orgId, type d'export |
| Changement statut registration | userId, orgId, registrationId, oldStatus → newStatus |
| Génération/impression badge | userId, orgId, attendeeId, badgeId |
| Actions sensibles attendees | userId, orgId, action, attendeeId |

---

## Plan d'action — Ordre d'exécution

```
1. Créer UserSystemAccess + migration Prisma
   └─ Migrer les is_root existants vers cette table

2. Supprimer is_root et is_platform de Role
   └─ Nettoyer le modèle Role

3. Modifier le JWT
   └─ Plus de mode: platform
   └─ Toujours : { sub, currentOrgId?, permissions?, isRoot? }

4. Modifier le login flow
   └─ Supprimer le branchement platformRole
   └─ Tenant : logique single-org / multi-org via OrgUser
   └─ Root (détecté via UserSystemAccess) : JWT sans org + liste de TOUTES les orgs, ignore OrgUser

5. Brancher les endpoints manquants
   └─ GET /auth/me/orgs
   └─ POST /auth/switch-org

6. Activer TenantContextGuard
   └─ Sur toutes les routes qui manipulent des données tenant

7. Audit service minimal
   └─ Login + switch-org + actions root

8. Tester tous les cas
   └─ Tenant single-org
   └─ Tenant multi-org + switch
   └─ Root + switch vers n'importe quelle org
   └─ User sans org → erreur propre
```

---

## Cible future — Platform access v2 (non implémenté)

> Documenté ici pour mémoire. Ne sera implémenté qu'en Phase 2/3.

Modèle cible pour les accès internes :
- Rôles internes dédiés (SUPPORT, OPS) séparés des rôles tenant
- Accès par assignation explicite à des orgs
- Accès temporaires avec expiration
- Audit complet de toutes les actions platform
