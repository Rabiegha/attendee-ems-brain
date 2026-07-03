# User Deletion in Multi-Tenant Context

> **Statut** : Analyse / Documentation. **Aucun code modifié.**
> **Périmètre** : `attendee-ems-back` (modules `users`, `auth`) + wording frontend
> **Branche backend de référence** : `feat/authz-m1-audit-service`
> **Auteur** : audit interne

---

## 1. Résumé du problème potentiel

`User` représente une **identité globale** (email + mot de passe). Avant la bascule multi-tenant, supprimer un utilisateur dans EMS = désactiver cette identité globale.

Aujourd'hui, l'architecture est tenant-first :

- `User` = identité globale réutilisable.
- `OrgUser` = **membership** (lien `user_id × org_id`) qui matérialise l'accès du user à une organisation. Cf. [prisma/schema.prisma](../../../attendee-ems-back/prisma/schema.prisma#L90-L108).
- Un user peut appartenir à **plusieurs organisations** simultanément.
- Les données métier (events, registrations, badges, invitations…) appartiennent à une **organisation**, pas au user.

Conséquence : quand un **admin d'une organisation** "supprime" un utilisateur depuis son interface, ce qu'il veut faire est *"retirer cette personne de mon organisation"*, **pas** *"désactiver cette identité dans tout EMS"*. Si l'action désactive le `User` globalement, on casse :

- son accès aux **autres organisations** où il est légitime ;
- ses sessions actives en cours sur ces autres orgs ;
- toute future connexion, même non liée à l'org qui a déclenché la suppression.

C'est donc un sujet **multi-tenant** qui doit agir sur la **membership** (`OrgUser`), pas sur le `User`.

---

## 2. État actuel observé

### 2.1 Modèle `User`

[prisma/schema.prisma](../../../attendee-ems-back/prisma/schema.prisma#L44-L76)

- Champs liés à la désactivation : **`is_active: Boolean @default(true)`**.
- **Pas de `deletedAt`**.
- **Pas de `disabledAt`**.
- **Pas de `status` enum**.
- Relations notables : `orgMemberships: OrgUser[]`, `userRoles: UserRole[]`, `refreshTokens: RefreshToken[]`, `registrations`, `print_jobs`, `partnerScans`.

### 2.2 Modèle de membership : `OrgUser`

[prisma/schema.prisma](../../../attendee-ems-back/prisma/schema.prisma#L90-L108)

- PK composite `[user_id, org_id]`.
- FK `user_id` → `User` avec `onDelete: Cascade`.
- FK `org_id` → `Organization` avec `onDelete: Cascade`.
- Champs : `joined_at`, `created_at`, `updated_at`.
- **Pas de `deletedAt`** sur la membership.
- **Pas de `is_active`** sur la membership.
- **Pas de champ `role` ni `status`** sur la membership : le rôle est porté par `UserRole` (table séparée).

> ⚠️ La membership ne supporte donc **pas nativement** le soft delete aujourd'hui. Toute action de retrait nécessiterait soit un `delete` dur sur `OrgUser`, soit l'ajout d'un champ (`deletedAt` ou `is_active`).

### 2.3 Controller `users`

[src/modules/users/users.controller.ts](../../../attendee-ems-back/src/modules/users/users.controller.ts)

Endpoints de suppression :

| Méthode | Route | Guards | Permission | Source `orgId` | Service appelé |
|---|---|---|---|---|---|
| `DELETE` | `/users/:id` | `JwtAuthGuard`, `RequirePermissionGuard` | `users.delete` | `req.user.currentOrgId` | `usersService.softDelete(id, orgId)` |
| `DELETE` | `/users/bulk-delete` | idem | `users.delete` | `req.user.currentOrgId` | `usersService.bulkDelete(ids, orgId)` |
| `POST`   | `/users/bulk-restore` | idem | `users.delete` | `req.user.currentOrgId` | `usersService.bulkRestore(ids, orgId)` |
| `DELETE` | `/users/permanent-delete` | idem | `users.delete` | `req.user.currentOrgId` | `usersService.permanentDelete(ids, orgId)` |

### 2.4 Service `users` — méthodes de suppression

[src/modules/users/users.service.ts](../../../attendee-ems-back/src/modules/users/users.service.ts)

**`softDelete(userId, orgId)`** (lignes 413-429) :

```ts
async softDelete(userId, orgId) {
  const existing = await this.findOne(userId, orgId);  // vérifie membership
  if (!existing) throw new NotFoundException('User not found');
  return this.prisma.user.update({
    where: { id: userId },
    data: { is_active: false },          // ← agit sur le USER GLOBAL
  });
}
```

**`bulkDelete(ids, orgId)`** (lignes 431-455) :

```ts
await this.prisma.user.updateMany({
  where: {
    id: { in: ids },
    is_active: true,
    orgMemberships: { some: { org_id: orgId } },   // filtre d'éligibilité
  },
  data: { is_active: false, updated_at: new Date() },
});
```

**`bulkRestore(ids, orgId)`** (lignes 457-476) : symétrique, repasse `is_active: true`.

**`permanentDelete(ids, orgId)`** (lignes 484-530) :
- Sélectionne les users `is_active: false` membres de l'org.
- Annule les FK nullable (`registration.checked_in_by`, `checked_out_by`, `badge.generated_by`).
- Supprime les `invitation` envoyées par ces users.
- `tx.user.deleteMany(...)` → **hard delete** du `User`. Cascade Prisma supprime `OrgUser`, `UserRole`, `RefreshToken` de **toutes** les orgs.

### 2.5 Contexte tenant `currentOrgId`

[src/auth/interfaces/jwt-payload.interface.ts](../../../attendee-ems-back/src/auth/interfaces/jwt-payload.interface.ts#L13-L20)

- `currentOrgId` est dans le **payload JWT**, défini à la génération du token (`auth.service.ts → generateJwtForOrg()`).
- Récupéré dans les controllers via `req.user.currentOrgId`.
- N'est pas re-validé contre la base à chaque requête (un token reste valide jusqu'à expiration même si la membership disparaît).

### 2.6 Refresh tokens

`auth.service.ts` expose `revokeAllUserSessions(userId)` (lignes 339-344) :

```ts
await this.prisma.refreshToken.updateMany({
  where: { userId, revokedAt: null },
  data: { revokedAt: new Date() },
});
```

**Cette méthode n'est appelée par aucun des flux de suppression utilisateur.** Sur `permanentDelete`, le cascade Prisma supprime physiquement les rows `RefreshToken`, mais sur `softDelete` / `bulkDelete`, **les sessions actives restent valides**.

### 2.7 Frontend — wording

[attendee-ems-front/src/features/users/ui/DeleteUserModal.tsx](../../../attendee-ems-front/src/features/users/ui/DeleteUserModal.tsx)

- Titre FR : **"Désactiver cet utilisateur"** ([locales/fr/users.json](../../../attendee-ems-front/src/shared/lib/i18n/locales/fr/users.json#L108-L110))
- Titre EN : **"Deactivate this user"**
- Messages : "L'utilisateur ne pourra plus se connecter / Ses données seront conservées / Vous pourrez le réactiver plus tard si nécessaire".

[attendee-ems-front/src/features/users/ui/PermanentDeleteUserModal.tsx](../../../attendee-ems-front/src/features/users/ui/PermanentDeleteUserModal.tsx)

- Titre FR : **"Supprimer définitivement"** — bouton identique.
- Message : "Cette action supprimera définitivement toutes les données de cet utilisateur".

API consommée : [usersApi.ts](../../../attendee-ems-front/src/features/users/api/usersApi.ts#L169-L210)
- `deleteUser` → `DELETE /users/:id`
- `bulkDeleteUsers` → `DELETE /users/bulk-delete`
- `permanentDeleteUsers` → `DELETE /users/permanent-delete`
- `bulkRestoreUsers` → `POST /users/bulk-restore`

---

## 3. Comportement actuel exact

Quand un admin de l'organisation `org_abc` clique sur "Désactiver l'utilisateur Alice" :

1. Le front appelle `DELETE /users/{alice_id}`.
2. Le guard JWT vérifie l'auth + `users.delete`.
3. `softDelete(alice_id, currentOrgId='org_abc')` :
   - **vérifie** qu'Alice possède une membership dans `org_abc` (filtre d'éligibilité OK) ;
   - **exécute** `UPDATE "User" SET is_active = false WHERE id = alice_id`.

### Effets réels

| Donnée | État |
|---|---|
| `User.is_active` d'Alice | `false` — **globalement** |
| Membership `OrgUser(alice, org_abc)` | **inchangée** (existe toujours) |
| Membership `OrgUser(alice, org_xyz)` | **inchangée** (existe toujours) |
| Accès d'Alice à `org_xyz` | **cassé** : la connexion est bloquée car `is_active = false` empêche probablement le login (à vérifier dans `auth.service`) |
| Refresh tokens d'Alice | **non révoqués** — Alice reste connectée tant que ses tokens ne sont pas expirés, sur **toutes** ses orgs |
| Données métier créées par Alice | conservées |

Sur `bulkDelete` : même chose en lot.
Sur `permanentDelete` (uniquement après désactivation) : hard delete `User`, cascade supprime toutes les `OrgUser` et `UserRole`, supprime les `RefreshToken`. Là encore, action **globale**, pas scope-orga.

---

## 4. Bug ou non-bug

**Bug confirmé pour l'usage admin-d'organisation (cas A).**

Preuves dans le code :

- `softDelete()` agit sur `User.is_active` (champ global), pas sur la membership. Cf. [users.service.ts#L420-L428](../../../attendee-ems-back/src/modules/users/users.service.ts#L420-L428).
- La vérification `findOne(userId, orgId)` est utilisée comme **gate d'autorisation** (l'admin doit avoir accès au user dans son org), mais l'**action** mutée est globale.
- `bulkDelete()` filtre via `orgMemberships.some({ org_id: orgId })`, mais la mutation `updateMany` touche `User.is_active`. Idem : autorisation scope-org, effet global.
- `permanentDelete()` : même schéma, et le hard delete cascade sur **toutes** les memberships, pas seulement celle de l'org appelante.

Conséquence pratique : un admin de `org_abc` peut **désactiver le compte global** d'un utilisateur qui sert aussi `org_xyz`. Cela viole l'isolation tenant pour l'accès aux autres orgs.

> **Aspect partiellement OK** : l'isolation côté autorisation (qui peut agir sur qui) est correctement scopée. C'est l'**effet** de l'action qui ne l'est pas.

> **Risque sécurité associé non bloquant ici** : `revokeAllUserSessions()` n'est pas appelée → un user désactivé garde ses sessions actives jusqu'à expiration. Doc séparée recommandée.

---

## 5. Comportement attendu recommandé

Quand un **admin d'organisation** retire un utilisateur depuis l'interface de gestion users de son organisation :

- ❌ Ne pas modifier `User.is_active`.
- ❌ Ne pas créer ni renseigner `User.deletedAt` (n'existe pas, et ne doit pas être utilisé ici).
- ❌ Ne pas supprimer physiquement le `User`.
- ❌ Ne pas anonymiser le `User`.
- ✅ Soft-delete (ou supprimer) la membership `OrgUser(user, currentOrgId)` uniquement.
- ✅ Préserver les autres memberships actives du même user.
- ✅ Préserver les données métier créées par ce user (registrations, badges générés, etc.).
- ✅ Empêcher tout accès futur du user à cette organisation.
- ✅ Idéalement : invalider les refresh tokens du user **pour le contexte de cette org** (ou tous ses tokens si on ne sait pas isoler par org — à arbitrer).

> Ce document **ne demande pas** l'implémentation maintenant. Il documente la cible pour un futur ticket.

---

## 6. Règle d'architecture cible

- Un **`User`** est une **identité globale**. Il vit indépendamment des organisations.
- Une **`OrgUser`** (membership) est l'**accès** d'un user à une organisation. Sa présence/absence (ou son flag actif) détermine si le user peut entrer dans cette org.
- Les **données métier** (events, registrations, badges, print jobs, invitations…) appartiennent à **l'organisation**, pas au user.
- Les champs **`created_by_id` / `updated_by_id` / `checked_in_by` / `generated_by`** sont des **traces d'audit**. Ils ne doivent **pas casser** quand un user quitte une organisation. Solutions admissibles :
  - Champ FK **nullable** + `onDelete: SetNull` (déjà le cas pour `checked_in_by`, `checked_out_by`, `generated_by`).
  - Sinon, conserver la FK et afficher "Utilisateur retiré" côté UI sans casser la jointure.

**Exemple attendu** après retrait par l'admin de `org_abc` :

```
User
  id: user_123
  email: test@example.com
  is_active: true            # inchangé
  deletedAt: null            # (champ n'existe pas, et inchangé en tout état)

OrgUser
  userId: user_123
  orgId: org_abc
  deletedAt: now()           # ou row supprimée

OrgUser
  userId: user_123
  orgId: org_xyz
  deletedAt: null            # toujours actif
```

Le user existe toujours. Il n'a plus accès à `org_abc`. Il garde l'accès à `org_xyz`.

---

## 7. Cas métier à distinguer

### Cas A — Admin org retire un user de son organisation

**Scope du futur ticket de correction "membership".**

Action attendue :
- Soft delete (ou suppression) de la membership `OrgUser(user, currentOrgId)`.
- Aucune mutation sur `User`.
- Aucun impact sur les autres orgs du user.
- Empêcher l'accès futur à cette org (validation `OrgUser` au login / au switch-org / dans le guard tenant).
- Gérer les **tokens actifs** : a minima, documenter que `currentOrgId` doit être revalidé à chaque requête (un JWT pointant vers une membership supprimée doit être rejeté). Sinon, révoquer les refresh tokens du user.

### Cas B — Super admin désactive un compte globalement

**Hors scope du futur ticket "membership".**

Action future possible :
- Bloquer la connexion globalement (`User.is_active = false` ou nouveau champ `disabledAt`).
- Conserver les données métier.
- Conserver les traces d'audit.
- Appeler `revokeAllUserSessions(userId)` pour invalider tous les refresh tokens.
- Endpoint dédié, idéalement `POST /admin/users/:id/disable` avec permission `users.disable.global` (ou rôle root).

### Cas C — User demande la suppression définitive de son compte (RGPD)

**Hors scope du futur ticket "membership".**

Action future possible :
- Désactiver le compte immédiatement.
- Invalider les sessions et refresh tokens.
- Retirer toutes les memberships actives.
- **Anonymiser** les données personnelles : email → `deleted-<uuid>@anonymized.local`, first_name/last_name → "Utilisateur supprimé", phone → null, etc.
- Conserver les entités métier appartenant aux organisations (registrations, events…), avec FK pointant vers le compte anonymisé pour ne pas casser l'historique.
- Endpoint dédié `DELETE /me` ou flux admin RGPD.

> ⚠️ Ce cas nécessite une **décision produit + juridique + technique séparée** (RGPD, conditions d'utilisation, durée de rétention, droits de suppression vs obligation comptable). Il ne doit **pas** être traité dans le ticket membership.

---

## 8. Risques à vérifier avant toute future correction

- Désactiver/supprimer un user encore actif dans une **autre organisation** → cas actuel, à corriger.
- Casser un parcours **multi-org** (user qui switche d'org).
- Casser les relations `createdById` / `updatedById` (vérifier nullable / `SetNull`).
- Perdre l'**historique des events** créés par un user retiré.
- Casser des entités liées : **registrations, imports, badges, print jobs, invitations, partner scans**.
- Laisser un **JWT actif** dont `currentOrgId` pointe vers une membership supprimée → besoin de revalidation côté guard tenant.
- **Confusion frontend** entre "désactiver le compte" et "retirer de l'organisation" (wording actuel = "Désactiver l'utilisateur").
- Comportement différent attendu pour **root / super admin** (ne doit pas pouvoir être retiré accidentellement).
- Cas du **dernier admin** d'une organisation : interdire son retrait, sinon l'org devient ingérable.
- Cas où la membership est **déjà soft-deleted** (idempotence).
- Cas où la membership n'existe pas (404 explicite).
- Tenter de retirer un user d'une org où l'admin n'est pas admin (403).

---

## 9. Proposition d'architecture cible future

À ne **pas** implémenter dans cette tâche.

### Modèle

- Ajouter sur `OrgUser` : `deletedAt: DateTime?` (ou `is_active: Boolean @default(true)` + `deactivatedAt`).
- Index `@@index([org_id, deletedAt])` pour les listings.

### Endpoints

Au choix lors du futur ticket :

- **Option 1 (clarté)** : nouvel endpoint dédié
  - `DELETE /organizations/:orgId/users/:userId` → retire la membership.
  - `:orgId` doit correspondre à `currentOrgId` (guard).
- **Option 2 (compatibilité)** : conserver `DELETE /users/:id` mais **changer sa sémantique** côté service pour agir sur `OrgUser(userId, currentOrgId)`. Plus risqué (compat front) sauf si on documente clairement.
- **Option 3 (orienté membership)** : `DELETE /org-users/:membershipId`.

### Service

- Vérifier que l'appelant est admin de `currentOrgId`.
- Vérifier que la membership existe et n'est pas déjà supprimée.
- Vérifier que le user n'est pas le **dernier admin** de l'org (règle à confirmer).
- Soft delete `OrgUser.deletedAt = now()` (ou `delete` dur si on décide de ne pas garder d'historique de membership).
- Émettre un événement d'audit (`org.user.removed`).
- Optionnel : révoquer les refresh tokens du user dont le `currentOrgId` du JWT = cette org.

### Guards / tenant context

- Le guard tenant doit, à chaque requête, **vérifier qu'une membership active existe** pour `(req.user.sub, req.user.currentOrgId)`. Aujourd'hui, ce contrôle semble manquant.
- Si la membership a disparu/été soft-deleted, → 401/403, forcer un refresh ou un switch-org.

### Pas dans ce flux

- Aucune mutation sur `User`.
- Aucune anonymisation.
- Aucun delete cascade vers les données métier.

---

## 10. Points techniques à auditer plus tard

- Confirmer s'il faut **ajouter `deletedAt`** ou **`is_active`** sur `OrgUser`, ou supprimer la row tout court (impact historique).
- Vérifier où le **rôle tenant** est stocké (`UserRole`) et comment il interagit avec la suppression de membership : faut-il aussi soft-delete les `UserRole` scopées à cette org ?
- Comportement actuel de `auth.service.login()` face à `is_active = false` (refus de login ?) — à confirmer.
- Validation de `currentOrgId` côté guard à chaque requête (existe-t-il un `TenantContextGuard` qui revalide en DB ?).
- Flux **switch-org** après retrait : que se passe-t-il si l'org disparaît de la liste du user ?
- Comportement frontend après suppression : redirection ? rafraîchissement du token ?
- **Wording UI** dans `users.json` (FR/EN) à revoir → "Retirer de l'organisation".
- Règle métier sur le **dernier admin** : existe-t-elle quelque part ? Sinon, à définir.
- Comportement pour les utilisateurs **root / système** (`UserSystemAccess`) — ne pas pouvoir les retirer.
- Audit log : capter `org.user.removed` avec `actorUserId`, `targetUserId`, `orgId`, `at`.

---

## 11. Tests à prévoir pour un futur ticket

Tests fonctionnels (e2e) :

1. User membre d'**une seule org** → retiré : la membership est soft-deleted, `User.is_active` inchangé, plus d'accès à l'org.
2. User membre de **plusieurs orgs** → retiré d'une seule : les autres memberships restent intactes.
3. **`User.is_active` reste `true`** après retrait (test explicite).
4. Le user retiré **ne peut plus se connecter** à cette org (login ou switch-org rejeté).
5. Le user retiré peut **toujours se connecter** à ses autres orgs.
6. Les **events** et autres entités créés par le user restent visibles dans l'org.
7. Les relations `created_by_id` / `updated_by_id` **ne cassent pas** (jointures OK).
8. Pas d'appel à une logique de suppression/anonymisation globale.
9. Cas où la membership **n'existe pas** → 404 propre.
10. Cas où la membership est **déjà soft-deleted** → 409 ou 204 idempotent (à arbitrer).
11. Admin d'une org tente de retirer un user d'une **autre org** → 403.
12. Cas du **dernier admin** d'une organisation → 409 explicite (si règle adoptée).
13. JWT du user retiré, encore valide, tentant d'accéder à cette org → 401/403.
14. JWT du user retiré, tentant d'accéder à une autre org où il est encore membre → 200.

Tests unitaires :
- `removeMembership(userId, orgId)` service method.
- Guard tenant revalidant la membership.

---

## 12. Frontend / wording

État actuel dans [attendee-ems-front/src/shared/lib/i18n/locales/fr/users.json](../../../attendee-ems-front/src/shared/lib/i18n/locales/fr/users.json) :

- `modal.deactivate_title` → **"Désactiver cet utilisateur"**
- `modal.permanent_delete_title` → **"Supprimer définitivement"**
- `modal.deactivate_confirm` → **"Êtes-vous sûr de vouloir désactiver l'utilisateur {{name}} ?"**

Ce wording renforce le bug : l'admin org pense désactiver un compte alors qu'on devrait lui parler de retrait d'organisation.

À terme (dans le ticket membership), wording cible :

- FR : **"Retirer de l'organisation"** / "Êtes-vous sûr de vouloir retirer {{name}} de cette organisation ?"
- EN : **"Remove from organization"** / "Are you sure you want to remove {{name}} from this organization?"

Un wording séparé devra exister pour le cas B (super admin) et le cas C (suppression de compte RGPD), mais ces écrans n'existent pas encore.

> Aucun fichier frontend n'est modifié dans cette tâche.

---

## 13. Recommandation finale

**Bug confirmé** : le flux actuel de suppression admin org agit sur l'identité globale (`User.is_active`) au lieu d'agir sur la membership (`OrgUser`).

### Futur ticket "membership delete" — à faire :

- Corriger la suppression côté organisation pour qu'elle agisse **uniquement** sur la membership `OrgUser(userId, currentOrgId)`.
- Ajouter le champ de soft delete sur `OrgUser` (ou décider du hard delete).
- Renforcer le guard tenant pour revalider la membership à chaque requête.
- Mettre à jour le wording UI : "Retirer de l'organisation".
- Tests e2e listés en section 11.

### À **ne pas** faire dans ce ticket :

- Suppression définitive du compte.
- Anonymisation des données.
- Suppression globale ou désactivation globale de `User`.
- Refactoring complet des endpoints `users`.
- Toucher aux flux super admin / RGPD.

### Chantier futur séparé :

- Définir un vrai flow **"delete my account"** (cas C) avec décision produit + juridique.
- Définir le flow **"désactivation globale par super admin"** (cas B).
- Anonymiser les données personnelles tout en conservant l'historique métier.
- Invalider proprement tokens et sessions.
- Tracer dans l'audit log toutes les actions de retrait / désactivation / suppression.

---

## 14. Références

- Code backend :
  - [src/modules/users/users.controller.ts](../../../attendee-ems-back/src/modules/users/users.controller.ts)
  - [src/modules/users/users.service.ts](../../../attendee-ems-back/src/modules/users/users.service.ts#L413-L530)
  - [src/auth/auth.service.ts](../../../attendee-ems-back/src/auth/auth.service.ts#L339-L344)
  - [src/auth/interfaces/jwt-payload.interface.ts](../../../attendee-ems-back/src/auth/interfaces/jwt-payload.interface.ts#L13-L20)
  - [prisma/schema.prisma](../../../attendee-ems-back/prisma/schema.prisma#L44-L108)
- Frontend (lecture seule) :
  - [attendee-ems-front/src/features/users/ui/DeleteUserModal.tsx](../../../attendee-ems-front/src/features/users/ui/DeleteUserModal.tsx)
  - [attendee-ems-front/src/features/users/ui/PermanentDeleteUserModal.tsx](../../../attendee-ems-front/src/features/users/ui/PermanentDeleteUserModal.tsx)
  - [attendee-ems-front/src/features/users/api/usersApi.ts](../../../attendee-ems-front/src/features/users/api/usersApi.ts#L169-L210)
  - [attendee-ems-front/src/shared/lib/i18n/locales/fr/users.json](../../../attendee-ems-front/src/shared/lib/i18n/locales/fr/users.json#L108-L110)
