# Refactor Authorization — V2 (chantier post-V1)

> **Statut** : conception. À implémenter après stabilisation et exploitation V1 en prod.
>
> **Périmètre** : tout ce qui est sorti du scope V1 + dette identifiée pendant V1 + suggestions structurelles. Ce document est le **point d'entrée unique** du chantier V2.
>
> **Voir aussi** :
> - V1 : [Refactore authorization v1 = tenant-first only + root override.md](./Refactore%20authorization%20v1%20%3D%20tenant-first%20only%20%2B%20root%20override.md)
> - Phase 2 (couche internal, sous-chantier de V2) : [Phase 2 — Comptes internes non-root.md](./Phase%202%20%E2%80%94%20Comptes%20internes%20non-root.md)
> - Diagnostic V1 : [AUTHZ_V1_DIAGNOSTIC_FINAL](../../../../../../attendee-context-hub/analyse/AUTHZ_V1_DIAGNOSTIC_FINAL.md)
> - ADR transversal : [adr-004-authz-v1](../../../../../../attendee-context-hub/context/decisions/adr-004-authz-v1.md)

---

## 0. Vue d'ensemble

V1 a livré le modèle **tenant-first + root override** (un seul `User`, accès tenant via `OrgUser` + `UserRole`, root via `UserSystemAccess.is_root`).

V2 a trois axes :

1. **Solder la dette V1** (cleanup back-compat, drift detection complète, selectors centralisés…).
2. **Construire la couche `internal`** (staff produit non-tenant et non-root, support / billing / onboarding) — c'est l'objet de [Phase 2 — Comptes internes non-root](./Phase%202%20%E2%80%94%20Comptes%20internes%20non-root.md).
3. **Hardening sécurité & plateforme** (audit DB, MFA, sessions, throttling, feature flags, EventAccess fin, etc.).

---

## A. Items reportés depuis V1

### A.1 Couche `internal` — cœur de Phase 2
Détails complets dans [Phase 2](./Phase%202%20%E2%80%94%20Comptes%20internes%20non-root.md). Aucun rôle interne n'est modélisable en V1 ; le besoin produit doit le rendre indispensable avant de construire les tables et permissions `platform.*`.

### A.2 `AuditService` — persistance DB
V1 livre :
- `AuditPort` (interface hexagonale) — [src/platform/authz/ports/audit.port.ts](../../../../../../attendee-ems-back/src/platform/authz/ports/audit.port.ts)
- `LoggerAuditAdapter` (impl par défaut, log Nest structuré) — [src/platform/authz/adapters/logging/logger-audit.adapter.ts](../../../../../../attendee-ems-back/src/platform/authz/adapters/logging/logger-audit.adapter.ts)
- Injecté dans `PermissionsGuard` qui audit chaque check (allowed / denied).

À ajouter en V2 :
- Table `audit_logs (id, user_id, actor_type [tenant|internal|root], action, resource_type, resource_id, org_id, before/after JSON, ip, user_agent, created_at)`.
- `PrismaAuditAdapter implements AuditPort` (substitution simple via DI).
- Audit aussi : `AuthService.login()`, `AuthService.switchOrg()`, et toute action où `isRoot=true` ET décision `allow` (toutes les actions root sont auditées).
- Endpoint `GET /admin/audit-logs` (filtres : user, org, action, période).
- Rétention : 90 j hot + cold storage (S3) pour ≥ 1 an (RGPD/SOC2).

### A.3 Drift detection — évolutions
V1 livre la drift detection web complète ([src/services/rootApi.ts](../../../../../../attendee-ems-front/src/services/rootApi.ts)) : code 403 typé, toast i18n, redirect `/auth/select-org`, refetchOnFocus, clearOrgContext, resetApiState. Côté mobile : `handleTenantDriftThunk` dans [auth.slice.ts](../../../../../../attendee-ems-mobile/src/store/auth.slice.ts).

À ajouter en V2 :

| Comportement | V1 | V2 |
|---|---|---|
| Toast "vous avez maintenant accès à X" (positif) | ❌ | ✅ |
| Force logout si liste orgs **vide** ET user non-root | 🟡 écran "Aucune org" + bouton logout (V1.1) | ✅ logout auto + notif |
| Force logout si user désactivé back-side | ❌ | ✅ |
| Push notification "vous avez été ajouté à X" | ❌ | ✅ (si infra push prête) |

### A.4 `OrganizationSwitcher` Header (web) — améliorations
Le composant est livré en V1 ([src/features/auth/components/OrganizationSwitcher.tsx](../../../../../../attendee-ems-front/src/features/auth/components/OrganizationSwitcher.tsx)). V2 : ajouter dot état (synced / drift / read-only), liste filtrable si > 5 orgs (root), badge "ROOT mode" permanent, raccourci clavier.

### A.5 `selectIsRoot` selector centralisé (web + mobile)
Aujourd'hui éparpillé via `user.roles.includes('ROOT')` ou `user.isRoot`. V2 : un seul selector dans chaque slice + un hook `useIsRoot()`.

### A.6 Cleanup back-compat (Palier 4)
- Retirer `mode` de la réponse `POST /auth/login`.
- Retirer `requiresOrgSelection` (déduire client-side de `organization == null`).
- Retirer `mode` de la réponse `POST /auth/switch-org`.
- Retirer `isPlatform: false` figé dans `GET /auth/me/orgs`.
- Retirer `AuthContext.isPlatform` du type `core/types.ts`.
- Supprimer JSDoc obsolète mentionnant `user.mode` dans `current-user.decorator.ts`.
- Migration SQL : drop le rôle `SUPER_ADMIN` orphelin.
- Drop ou réécrire le trigger `trigger_check_tenant_role` (mentionne encore `platform_user_roles`).

### A.7 Permissions matrix figée (snapshot test)
Permissions chargées depuis DB. V2 : test snapshot freezant la matrice rôle → permissions en TS + script de diff DB ↔ code (drift detection infra). Empêche un seed mal écrit de bypass un test E2E.

### A.8 Rate limiting / Throttling sur auth
Aucun `ThrottlerGuard` sur `POST /auth/login`. V2 : 10 tentatives / 5 min / IP + lockout exponentiel par compte (5 échecs = 15 min, 10 = 1 h).

### A.9 2FA / MFA
Hors scope V1. V2 ou V3 : TOTP (Google Authenticator) pour les comptes root + admins. WebAuthn pour les internal.

### A.10 Session management
- Liste des sessions actives (refresh tokens valides) par user.
- `DELETE /auth/sessions/:id` pour révocation chirurgicale.
- `POST /auth/sessions/revoke-all` pour "déconnexion partout".
- Affichage IP / user-agent / dernière utilisation par session.

### A.11 Impersonation contrôlée (root uniquement)
- `POST /admin/impersonate { userId }` → JWT avec `impersonatedBy: <root_user_id>`.
- Audit obligatoire (entrée + sortie + toutes actions taggées impersonation).
- TTL court (15 min max).
- UI : bandeau rouge permanent "Connecté en tant que X — sortir".

### A.12 Multi-org switch côté print client (Electron)
`attendee-ems-print-client` n'a pas de notion d'org. V2 : si un poste sert plusieurs orgs (centre de conférences), prévoir compte technique par org OU session multi-org sur le client.

### A.13 Org-level feature flags
Permissions binaires aujourd'hui. V2 : `OrgFeature (org_id, feature_code, enabled, plan_tier)` pour activer/désactiver des modules entiers (badge-printing, partner-scans, n8n) selon la souscription. RBAC vérifie permission **ET** feature activée.

### A.14 EventAccess (scope `assigned`)
`permission.scope = 'assigned'` existe conceptuellement mais n'est pas matérialisée. V2 : table `event_access (user_id, event_id, role_in_event)` pour limiter un user à certains events de son org.

### A.15 Org-level audit dashboard (UI admin)
Vue web pour les admins d'une org : qui a fait quoi, quand, sur quelle ressource. Filtrable. Exportable CSV.

---

## B. Suggestions structurelles V2

### B.1 Découpler `Role` métier de `Permission` machine
Aujourd'hui : `Role` ↔ `Permission` via `RolePermission` (M:N). V2 : ajouter une couche `Capability` (groupes nommés de permissions), ex: `Capability:event-organizer` = `events.create + events.update + attendees.import + …`. Les `Role` composent des Capabilities.

### B.2 Test contractuel back ↔ clients (web + mobile)
Le bug "back camelCase, mobile snake_case" doit ne jamais réapparaître. V2 :
- OpenAPI/Swagger généré automatiquement par NestJS sur les endpoints critiques.
- Génération TypeScript front + mobile (openapi-typescript ou orval).
- CI fail si la shape change sans bump de version.

### B.3 Versionner l'API auth
Préfixer `/v2/auth/*` quand V2 sortira, garder `/v1/auth/*` 6 mois. Permet de droper `mode` / `isPlatform` proprement.

### B.4 Domain Events sur les actions auth
Émettre `UserLoggedIn`, `UserSwitchedOrg`, `RoleAssigned`, `RootBypassUsed` pour :
- Audit asynchrone (Kafka / Postgres LISTEN-NOTIFY).
- Hooks externes (n8n, Slack alerts).
- Rebuild d'analytics sans coupler les services.

### B.5 Repenser `Public-Form-Logger` ↔ Auth
PFL a sa propre auth séparée. V2 : compte technique `service.public-form-logger` côté EMS avec rôle `INTEGRATION_LIMITED` et accès JWT m2m (`client_credentials`).

### B.6 Tests de mutation sur le RBAC
Ajouter Stryker (mutation testing) sur `AuthorizationService.can()` et `verifyOrgAccess()` — les bugs d'authorization sont catastrophiques.

---

## C. Priorisation V2 suggérée

| Phase V2 | Items | Effort |
|---|---|---|
| **V2.0 — Cleanup & dette V1** | A.6, A.5, A.4, A.3 | 2 sem |
| **V2.1 — Audit & Compliance** | A.2, A.8, A.10 | 3 sem |
| **V2.2 — Couche internal** | Tout [Phase 2](./Phase%202%20%E2%80%94%20Comptes%20internes%20non-root.md) + A.11 | 6-8 sem |
| **V2.3 — Features org-level** | A.13, A.14, A.15 | 4 sem |
| **V2.4 — Sécurité avancée** | A.9, B.6, A.7 | 3 sem |
| **V2.5 — Plateforme** | B.1, B.2, B.3, B.4, B.5 | 6 sem |

---

## D. Décisions à prendre avant V2

1. **Niveau de compliance visé ?** SOC2 ? RGPD strict ? Détermine la rigueur d'audit (A.2) et de session management (A.10).
2. **Priorité business : feature flags (A.13) ou impersonation (A.11) ?** L'un est revenue, l'autre est support.
3. **Mobile parity-first ou web-first ?** A.4 web-only, A.3 mobile-first (déjà en cours).
4. **CASL côté back ?** Le front l'utilise via `/auth/policy`. Côté back encore `RequirePermissionGuard` ad-hoc. Unifier ?
5. **Pluggable identity providers (SSO, SAML, OIDC) ?** Pas dans Phase 2, mais à anticiper dans le design des `User`.

---

## E. Anti-patterns à éviter en V2

- ❌ Re-mélanger root et tenant dans le même type `User` sans discriminant.
- ❌ Hardcoder des rôles en string dans le code applicatif (toujours via constantes typées).
- ❌ Donner à l'`internal` GLOBAL un accès silencieux sans audit.
- ❌ Permettre à un internal de **créer** des données tenant (lecture / support / setup uniquement).
- ❌ Stocker les permissions effectives dans un JWT de session longue (laisser revalider via `/auth/policy`).
- ❌ Cacher les permissions côté front sans TTL → drift garanti.
