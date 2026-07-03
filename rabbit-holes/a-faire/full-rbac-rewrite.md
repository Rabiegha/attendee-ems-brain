# Rabbit Hole — Full RBAC rewrite

Type: Rabbit Hole
Status: Watchlist
Risk level: High

## 1. Why it is tempting

- Le RBAC actuel a des coins difficiles (scopes `any/org/assigned/none/own`).
- Le projet utilise `@casl/ability` + une couche hexagonale `platform/authz/` → tentation de tout repenser pour être « plus pur ».
- Un travail de refonte RBAC est en cours sur d'autres repos (branches `Refactore-authorization-v1`, `feat/authz-m1-audit-service`, `feat/authz-m5-drift-detection`) → on peut être tenté de tout aligner d'un coup.
- Plusieurs analyses existent (voir [../../analyse/AUTHZ_V1_DIAGNOSTIC_FINAL.md](../../../attendee-context-hub/analyse/AUTHZ_V1_DIAGNOSTIC_FINAL.md), [docs/Refactore-autorization/](../../backlog/a-faire/other/migrations and refactoring/Refactore-autorization)).

## 2. Why it is dangerous now

- Le RBAC **touche tous les endpoints** : refonte = risque de régression sécurité massive.
- Une refonte mal testée = élévation de privilèges / cross-tenant.
- Le focus actuel est **Async Architecture** ; refondre le RBAC en parallèle disperserait l'équipe.
- Le travail RBAC déjà en cours (audit-service, drift-detection) doit d'abord **converger** avant qu'on touche aux primitives.
- Aucune métrique objective ne dit « le RBAC actuel ne tient pas la charge » — on parlerait d'élégance, pas de besoin.

## 3. Scope creep risk

Très élevé :
- Touche guards, décorateurs, services, controllers, frontend (UI policies), mobile.
- Modifier le format des permissions casse les tokens en vol.
- Forcer une migration des rôles tenant ⇒ casse client.
- Risque de réinventer CASL en wrapper maison.

## 4. Current decision

- **Pas maintenant.**
- Le RBAC actuel reste en place.
- Le chantier `feat/authz-m1-audit-service` continue **dans son scope strict** (audit), pas comme rampe de lancement d'une refonte totale.
- Le drift detection (`feat/authz-m5-drift-detection`) reste un outil d'observation, pas un déclencheur de refonte.

## 5. When to revisit

- Quand un cas métier prouvé montre que le modèle actuel **bloque** une fonctionnalité (pas juste « inélégant »).
- Quand les chantiers RBAC en cours (audit, drift) seront mergés et stables.
- Avec un plan de migration progressif, jamais en big-bang.

## 6. Related docs

- [docs/Refactore-autorization/](../../backlog/a-faire/other/migrations and refactoring/Refactore-autorization)
- [attendee-context-hub/analyse/AUTHZ_V1_DIAGNOSTIC_FINAL.md](../../../attendee-context-hub/analyse/AUTHZ_V1_DIAGNOSTIC_FINAL.md)
- [attendee-context-hub/analyse/AUTHZ_V1_DRIFT_DETECTION.md](../../../attendee-context-hub/analyse/AUTHZ_V1_DRIFT_DETECTION.md)
- [attendee-context-hub/analyse/AUTHZ_V1_SMOKE_TESTS.md](../../../attendee-context-hub/analyse/AUTHZ_V1_SMOKE_TESTS.md)
- [../async-architecture/QandA/09-security-multitenancy.md](../../workstreams/a-faire/async-architecture/QandA/09-security-multitenancy.md)

## 7. Rule for Copilot

- Si une proposition contient « refondre le RBAC », « réécrire les guards », « remplacer CASL », « centraliser les permissions autrement » → **stop**, pointer vers ce document.
- Autorisé : améliorations **locales et tickétées** (ex : ajouter une permission manquante sur un endpoint précis).
- Interdit : modifier `JwtAuthGuard`, `RequirePermissionGuard`, `PermissionsGuard`, le contenu du JWT, ou la table des rôles sans ticket explicite et review humaine.
