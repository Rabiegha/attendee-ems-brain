# Tests automatisés — Unitaires, fonctionnels, E2E, intégration

> **Objectif :** mettre en place une suite de tests qui garantisse la non-régression pour TOUT futur changement sur l'authz, pas seulement ce refactor.
> **Quand :** une fois le refactor front + mobile terminé, **avant** Palier 4.

## Pourquoi maintenant et pas avant

Faire les tests automatisés AVANT le refactor aurait demandé d'écrire les tests sur l'ancien système, puis de tout réécrire. On les fait MAINTENANT pour figer le nouveau comportement et détecter toute régression future.

## Pyramide des tests cible

```
                  ▲
                 ╱ ╲   E2E Playwright (web) + Detox (mobile)
                ╱___╲  ~10 scénarios critiques
               ╱     ╲
              ╱       ╲   Intégration API (supertest + DB test)
             ╱─────────╲  ~30 endpoints
            ╱           ╲
           ╱             ╲   Unitaires (Jest)
          ╱_______________╲  ~150 tests sur Core RBAC
```

---

## A. Tests unitaires Back (Jest)

### A.1 — Core RBAC (priorité haute)

**Fichiers cibles :**

- `src/platform/authz/core/authorization.service.spec.ts`
- `src/platform/authz/core/permission-resolver.spec.ts`
- `src/platform/authz/core/scope-evaluator.spec.ts`
- `src/platform/authz/core/decision.spec.ts`

**Cas à couvrir pour `AuthorizationService.can()` :**

- [ ] User root → bypass, allow systématique
- [ ] User sans rôle dans l'org cible → deny `NO_ROLE`
- [ ] User avec rôle mais sans la permission demandée → deny `NO_PERMISSION`
- [ ] User avec permission scope `any` → allow
- [ ] User avec permission scope `org` → allow si même org
- [ ] User avec permission scope `assigned` + resource assignée → allow
- [ ] User avec permission scope `assigned` + resource non assignée → deny `SCOPE_MISMATCH`
- [ ] User avec permission scope `own` + resource owner = user → allow
- [ ] User avec permission scope `own` + resource owner ≠ user → deny `SCOPE_MISMATCH`
- [ ] Cross-org : user de Choyou tente une action sur Acme sans bypass → deny

### A.2 — Adapters Prisma

**Fichiers cibles :**

- `src/platform/authz/adapters/db/prisma-rbac-query.adapter.spec.ts`
- `src/platform/authz/adapters/db/prisma-membership.adapter.spec.ts`
- `src/platform/authz/adapters/db/prisma-auth-context.adapter.spec.ts`

Mock `PrismaService`, vérifier les bons appels (where clauses, includes).

### A.3 — Auth service

**Fichier :** `src/auth/auth.service.spec.ts`

- [ ] `login` : credentials valides → token
- [ ] `login` : password incorrect → UnauthorizedException
- [ ] `getAvailableOrgs` pour root → 3 orgs (memberships + bypass)
- [ ] `getAvailableOrgs` pour user normal → uniquement ses memberships
- [ ] `switchOrg` : org valide → nouveau token
- [ ] `switchOrg` : org où l'user n'a pas accès et pas root → ForbiddenException
- [ ] `getUserAbility` root sans rôle dans l'org → 200 synthétique (régression Palier 3c)

### A.4 — Guards

- [ ] `JwtAuthGuard` : sans token → 401
- [ ] `PermissionsGuard` : permission OK → next
- [ ] `PermissionsGuard` : permission KO → ForbiddenException
- [ ] `RequirePermissionGuard` : décorateur `@Public` → bypass

---

## B. Tests d'intégration Back (Jest + supertest + DB de test)

**Setup :** DB Postgres dédiée tests (`ems_test`), reset entre chaque suite.

**Fichier suggéré :** `test/integration/auth-flow.e2e-spec.ts`

```ts
describe('Auth flow integration', () => {
  beforeAll(async () => {
    // Setup DB test, seed users
  });

  it('full auth flow: login → me/orgs → switch-org → me/ability', async () => {
    const login = await request(app).post('/auth/login').send({...});
    const orgs = await request(app).get('/auth/me/orgs').set('Authorization', `Bearer ${login.body.access_token}`);
    const switched = await request(app).post('/auth/switch-org').send({...});
    const ability = await request(app).get('/me/ability').set('Authorization', `Bearer ${switched.body.accessToken}`);
    expect(ability.body.grants).toBeDefined();
  });
});
```

Endpoints à couvrir minimum :
- [ ] `POST /auth/login`
- [ ] `POST /auth/refresh`
- [ ] `POST /auth/logout`
- [ ] `GET /auth/me/orgs`
- [ ] `POST /auth/switch-org`
- [ ] `GET /auth/policy`
- [ ] `GET /me/ability`
- [ ] `GET /organizations` (test bypass root)
- [ ] `POST /organizations` (test bypass root + restriction non-root)
- [ ] `GET /events` (test scope `assigned`)

---

## C. Tests E2E Web (Playwright)

Cf. `04-TESTS-E2E-INTEGRATION.md` section C.

À pérenniser dans `attendee-ems-front/tests/e2e/auth-refactor-v1.spec.ts`.

---

## D. Tests E2E Mobile (Detox)

Si Detox configuré dans `attendee-ems-mobile`, ajouter `e2e/authRefactor.test.ts`.
Sinon, documenter le smoke test manuel comme procédure récurrente.

---

## E. CI

Mettre tout cela dans la pipeline CI :

```yaml
# .github/workflows/ci.yml (back)
jobs:
  test-back:
    steps:
      - run: npm test                    # unitaires
      - run: npm run test:e2e            # intégration supertest
  test-front:
    steps:
      - run: npm test                    # unitaires
      - run: npm run test:e2e            # Playwright headless
```

Bloquer le merge si rouge.

---

## Checklist d'acceptation

- [ ] Couverture Core RBAC > 80%
- [ ] Couverture AuthService > 70%
- [ ] Au moins 1 test d'intégration par endpoint critique
- [ ] Au moins 5 tests Playwright e2e couvrant les flux principaux
- [ ] CI passe en < 5 minutes
- [ ] Tous les tests verts sur la branche `Refactore-authorization-v1`
