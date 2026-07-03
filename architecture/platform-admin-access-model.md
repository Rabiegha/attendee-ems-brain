# Platform Admin Access Model

> **Statut** : Documentation architecture / Cible. **Aucun code modifié.**
> **Public** : Architecture, Sécurité, Dev backend & frontend

---

## 1. Objectif

Documenter la **séparation stricte** entre deux espaces d'accès :

- **Tenant access** : ce qu'un user peut faire **à l'intérieur d'une organisation** (mes events, mes attendees, mes badges…).
- **Platform access** : ce qu'un user peut faire **au niveau de la plateforme Attendee** (lister toutes les orgs, gérer les plans, support client, audit).

Ces deux espaces doivent avoir des **rôles distincts**, des **guards distincts** et des **routes distinctes**.

---

## 2. Problème à éviter

**Ne pas faire d'un Platform Admin un "Owner magique" implicite de toutes les organisations.**

Risques d'un tel raccourci :

- **Confusion RBAC** : les permissions tenant (`events.create`, `users.invite`…) ne sont pas pensées pour des actions cross-org. Les autoriser globalement crée des angles morts.
- **Accès accidentel aux données tenant** : un Platform Admin qui navigue dans une org pour debug peut consulter / modifier des données sensibles sans trace explicite.
- **Bypass des scopes org** : `currentOrgId` perd son sens si un user peut tout voir ; le filtrage par tenant devient optionnel et donc oubliable.
- **Mauvaise séparation sécurité** : compromission d'un compte Platform Admin = compromission de **toutes** les orgs.
- **Audit compliqué** : impossible de distinguer "action légitime d'Owner" vs "action de Platform Admin pour support".
- **Risque RGPD/contractuel** : les clients s'attendent à ce que leurs données ne soient consultables que sous conditions explicites (impersonation tracée, ticket support…).

> Règle : **un Platform Admin n'a aucune permission tenant par défaut**. Pour intervenir dans une org, il doit passer par un mécanisme explicite (lecture seule plateforme, ou impersonation contrôlée et tracée — à décider plus tard).

---

## 3. Modèle recommandé

### 3.1 Tenant roles

Liés à une **membership** (`OrgUser`) :

- **Owner** — créateur de l'org, droits maximum tenant, gère billing owner.
- **Admin** — gère users, paramètres, events.
- **Manager** — gère events et opérations sans toucher aux users/paramètres org.
- **Staff** — accès opérationnel (check-in, badges) sans gestion.

Les permissions tenant existantes (`events.*`, `users.*`, etc.) restent **scopées à l'organisation** via `currentOrgId`.

### 3.2 Platform roles

**Indépendants** de toute organisation, **indépendants** de `currentOrgId` :

- **Platform Admin** — accès complet plateforme (orgs, users globaux, plans, suspension, audit).
- **Platform Support** — lecture des orgs/users pour aider, actions limitées (renvoyer invitation, reset password global).
- **Platform Billing Manager** — gestion plans, subscriptions, factures, trial extensions, refunds.

Un user peut avoir **0, 1 ou plusieurs** rôles plateforme. Avoir un rôle plateforme **ne donne aucun droit tenant** automatiquement.

---

## 4. Architecture cible (conceptuelle)

```
User
├── tenantMemberships: Membership[]      ← rôles tenant (par org)
│     ├── (orgId: org_a, role: Owner)
│     └── (orgId: org_b, role: Staff)
└── platformAccess: PlatformAccess?      ← rôles plateforme (global)
      └── roles: [PlatformAdmin]
```

### Exemple

```
User Rabie
├── Org A : Owner
├── Org B : Staff
└── Platform : Platform Admin
```

Interprétation :
- Dans **Org A**, Rabie agit comme Owner via JWT `currentOrgId=org_a`.
- Dans **Org B**, Rabie agit comme Staff via JWT `currentOrgId=org_b`.
- Sur la route `/platform`, Rabie agit comme Platform Admin **sans** `currentOrgId` requis (ou avec un `currentOrgId` ignoré par le platform guard).

> Important : `Platform Admin` **n'apparaît pas** comme un rôle dans `Org A` ni dans `Org B`. Le switcher d'org ne le mentionne pas.

---

## 5. Guards recommandés (conceptuel)

À documenter pour le futur ticket, **non implémenté ici**.

| Guard | Vérifie | S'applique à |
|---|---|---|
| `JwtAuthGuard` | JWT valide | Toutes les routes authentifiées |
| `TenantGuard` | `currentOrgId` présent + membership existante et active | Routes tenant (`/api/events`, `/api/users`, …) |
| `RequirePermissionGuard` | permission tenant (`events.create`, …) sur `currentOrgId` | Routes tenant à permission |
| `PlatformGuard` | user possède un rôle plateforme + rôle requis | Routes `/api/platform/*` |

Règles clés :

- **Routes plateforme** : passent par `JwtAuthGuard` + `PlatformGuard`, **jamais** par `TenantGuard`.
- **`currentOrgId` n'est pas requis** sur les routes plateforme. S'il est présent dans le JWT, il est ignoré ou utilisé comme simple hint UI.
- **Aucun fallback** : un Platform Admin qui tente une route tenant sans membership doit recevoir 403, **pas** 200.
- **Impersonation** (si décidée plus tard) : nouveau guard `ImpersonationGuard` + middleware d'audit qui force l'écriture dans audit log avec `actorUserId` réel + `impersonatedUserId`.

---

## 6. Route frontend recommandée

### Pour maintenant

- **Même frontend** `attendee-ems-front`. Pas de nouveau projet.
- Route séparée **`/platform`** à la racine, distincte de l'arborescence tenant (`/dashboard`, `/events`, …).
- **Navigation visible uniquement** si l'utilisateur a un rôle plateforme (lien "Platform" dans le menu profil par exemple).
- **Layout dédié** `/platform` : pas de sélecteur d'org tenant, pas de filtrage par `currentOrgId` dans l'UI.
- **Bundle séparé optionnel** (code splitting Vite) pour éviter de charger les pages platform aux users non-platform.

### Pour plus tard (évolution possible)

- Extraction vers un sous-domaine dédié `admin.attendee.fr` si la surface plateforme grandit suffisamment.
- Authentification renforcée (2FA obligatoire) sur le sous-domaine plateforme.
- Logs d'accès séparés.

---

## 7. Fonctionnalités Platform Admin futures

Liste indicative pour cadrer le scope :

**Organisations**
- Voir la liste de toutes les organisations.
- Voir le plan, le statut (active / suspended / archived), la date de création, le créateur.
- Changer manuellement le plan d'une org (override).
- Appliquer / prolonger un Trial.
- Suspendre / réactiver une organisation.
- Archiver une organisation.

**Users**
- Voir tous les users globaux.
- Voir les memberships d'un user.
- Désactiver / réactiver un compte global (cas B de la doc user-deletion).
- Reset password (envoi email).
- Voir les sessions actives et révoquer.

**Plans & Billing**
- CRUD des plans (features, limites).
- Voir les subscriptions, leurs status, leur historique.
- Gérer billing owner d'une org.

**Audit & Support**
- Audit logs filtrables par org / user / action.
- Notes de support attachées à une org.
- **Impersonation contrôlée** — uniquement si validée plus tard (validation produit + juridique + sécurité), avec :
  - bouton explicite ;
  - durée limitée (15 min) ;
  - audit log obligatoire avec raison saisie ;
  - bandeau visible "Vous agissez en tant que X" ;
  - actions write potentiellement bloquées (à arbitrer).

---

## 8. Recommandation finale

- **Pas de nouveau frontend** maintenant. Garder `attendee-ems-front`.
- Route **`/platform`** dans le frontend existant, layout dédié.
- Rôles plateforme **séparés** des rôles tenant.
- Guards **séparés** : `PlatformGuard` ≠ `TenantGuard` + `RequirePermissionGuard`.
- Un Platform Admin **n'hérite d'aucun droit tenant**.
- **Impersonation = chantier séparé**, désactivé tant que non validé.
- L'audit log doit distinguer **actions tenant** vs **actions plateforme**.
