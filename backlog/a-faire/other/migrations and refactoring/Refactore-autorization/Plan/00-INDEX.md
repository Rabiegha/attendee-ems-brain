# Plan global — Refactor Authorization v1

> **Source de vérité du chantier complet, du code au deploy prod.**
> **Branche :** `Refactore-authorization-v1` (back, front, mobile)

## Etat actuel (24 avril 2026)

| Phase | Statut | Commit/PR | Notes |
|---|---|---|---|
| Palier 1 — UserSystemAccess + data migration | ✅ FAIT | (commits antérieurs) | Migration des roots faite localement |
| Palier 2 — Refactor code authz | ✅ FAIT | (commits antérieurs) | freeze isPlatform=false |
| Palier 3a — Drop platform (DB + code) | ✅ FAIT | commit antérieur | DROP TABLE platform_*, columns is_platform/is_root |
| Palier 3b — Rename TenantUserRole → UserRole | ✅ FAIT | `59e2bdd` | Tables, types, ports |
| Palier 3c — Cleanup back (bug + logs + commentaire) | ✅ FAIT | `c6d8e7a` | Fix /auth/policy, Logger.debug |
| Refactor Front | ⏳ À FAIRE | — | Voir [02-REFACTOR-FRONT.md](02-REFACTOR-FRONT.md) |
| Refactor Mobile | ⏳ À FAIRE | — | Voir [03-REFACTOR-MOBILE.md](03-REFACTOR-MOBILE.md) |
| Tests E2E intégration | ⏳ À FAIRE | — | Voir [04-TESTS-E2E-INTEGRATION.md](04-TESTS-E2E-INTEGRATION.md) |
| Tests unitaires/fonctionnels | ⏳ À FAIRE | — | Voir [06-TESTS-UNITAIRES-FONCTIONNELS-E2E.md](06-TESTS-UNITAIRES-FONCTIONNELS-E2E.md) |
| Plan de déploiement | 📋 PRÊT | — | Voir [07-PLAN-DEPLOIEMENT.md](07-PLAN-DEPLOIEMENT.md) |
| Tests post-prod | 📋 PRÊT | — | Voir [08-TESTS-POST-PROD.md](08-TESTS-POST-PROD.md) |
| Palier 4 — Retrait HTTP deprecated | ⏳ À FAIRE après deploy | — | Voir [05-PALIER-4-cleanup-http-deprecated.md](05-PALIER-4-cleanup-http-deprecated.md) |

## Sommaire des fichiers

1. [01-PALIER-3C-cleanup-back.md](01-PALIER-3C-cleanup-back.md) — Cleanup back (✅ fait)
2. [02-REFACTOR-FRONT.md](02-REFACTOR-FRONT.md) — Plan refactor web
3. [03-REFACTOR-MOBILE.md](03-REFACTOR-MOBILE.md) — Plan refactor mobile
4. [04-TESTS-E2E-INTEGRATION.md](04-TESTS-E2E-INTEGRATION.md) — Tests bout-en-bout local/staging
5. [05-PALIER-4-cleanup-http-deprecated.md](05-PALIER-4-cleanup-http-deprecated.md) — Retrait final des champs HTTP dépréciés
6. [06-TESTS-UNITAIRES-FONCTIONNELS-E2E.md](06-TESTS-UNITAIRES-FONCTIONNELS-E2E.md) — Suite de tests automatisés
7. [07-PLAN-DEPLOIEMENT.md](07-PLAN-DEPLOIEMENT.md) — Procédure deploy complète (back + front + mobile + Palier 4)
8. [08-TESTS-POST-PROD.md](08-TESTS-POST-PROD.md) — Vérifications post-deploy en prod

## Ordre d'exécution recommandé

```
[FAIT] Paliers back 1, 2, 3a, 3b, 3c
   ↓
[ICI] Refactor Front + Mobile (en parallèle, branches séparées)
   ↓
Tests E2E + tests automatisés (sur les 3 repos)
   ↓
Validation locale + staging complète
   ↓
Backup prod + Deploy back + migration DB + script root
   ↓ (24h obs)
Deploy front
   ↓ (24h obs)
Deploy mobile (OTA ou store)
   ↓ (7j obs)
Palier 4 — Retrait champs HTTP dépréciés
   ↓
Tests post-prod finaux + clôture chantier
```

## Principe directeur

> **On ne déploie qu'une fois TOUT validé bout-en-bout.**
> Pas de half-deploy, pas de deploy partiel "pour voir".
