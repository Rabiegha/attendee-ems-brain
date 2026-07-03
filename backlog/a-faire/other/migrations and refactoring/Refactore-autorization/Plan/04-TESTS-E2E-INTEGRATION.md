# Tests bout-en-bout (E2E) — Local & Staging

> **Quand exécuter :** une fois le refactor front + mobile terminés, **avant** le Palier 4 back et **avant** le déploiement.
> **Où :** environnement local d'abord, puis staging si dispo, puis pré-prod.

## Pré-requis

- [ ] Back local sur `Refactore-authorization-v1` à jour (Palier 3c minimum)
- [ ] Front local sur `Refactore-authorization-v1` à jour
- [ ] Mobile local sur `Refactore-authorization-v1` à jour
- [ ] DB locale fraîchement seedée (script `scripts/seed-local.sh` ou équivalent)
- [ ] Au moins 4 comptes utilisateurs préparés :
  - 🔴 **root** : `rgharghar@choyou.fr` (UserSystemAccess.is_root = true)
  - 🟠 **admin org Choyou** : un user avec rôle Administrator dans Choyou
  - 🟡 **agent org Choyou** : un user avec rôle Agent dans Choyou
  - 🟢 **user multi-org** : un user membre de Choyou ET Acme avec rôles différents

## Stratégie

| Niveau | Outil | Quand |
|---|---|---|
| **Smoke manuel** | Browser + simulateur mobile | Systématique avant tout déploiement |
| **E2E automatisé web** | Playwright | Sur chaque PR + avant deploy |
| **E2E automatisé mobile** | Detox (si configuré) | Avant deploy |
| **Test API isolé** | Script curl + Postman/Bruno | Validation back seul |

---

## A. Smoke tests manuels web

### A.1 — Authentification

- [ ] Login avec un user existant → JWT reçu, redirection sur dashboard de l'org par défaut
- [ ] Login avec mauvais mot de passe → 401, message d'erreur clair
- [ ] Login avec email inexistant → 401, **pas** d'info sur l'existence du compte
- [ ] Logout → token effacé, redirection sur /login
- [ ] Refresh token automatique avant expiration JWT

### A.2 — Switch d'organisation

- [ ] User multi-org : la liste des orgs s'affiche correctement (3 orgs pour root)
- [ ] Switch sur Acme : nouveau JWT, nouvelle org dans le header, données rechargées
- [ ] Switch sur une org où le user n'a aucune permission → UI gracieuse (pas de plantage)
- [ ] Refresh page après switch → on reste sur la bonne org

### A.3 — Bypass root

- [ ] Login en tant que root → switch sur Acme (où root n'a pas de membership)
- [ ] Lister les organisations → 200 avec toutes les orgs (bypass actif)
- [ ] Créer une organisation → autorisé
- [ ] Lister les attendees de l'événement Acme → autorisé sans membership

### A.4 — RBAC normal

- [ ] Login admin Choyou → accès complet aux pages Choyou
- [ ] Admin Choyou tente d'accéder à Acme → 403 ou redirection
- [ ] Agent Choyou : voit uniquement les events qui lui sont assignés (scope `assigned`)
- [ ] Agent Choyou tente de créer un event → 403 (pas la perm)

### A.5 — Endpoints critiques

- [ ] `GET /auth/me/orgs` → liste correcte
- [ ] `POST /auth/switch-org` → token rafraîchi
- [ ] `GET /auth/policy` → 200 même pour root sans rôle dans l'org
- [ ] `GET /me/ability` → grants corrects pour l'org active

---

## B. Smoke tests manuels mobile

Reprendre A.1 → A.5 sur :
- [ ] iOS (au moins un device réel)
- [ ] Android (au moins un device réel)

Plus, spécifique mobile :
- [ ] App killée puis relancée → session restaurée depuis secureStore
- [ ] Mode avion → erreur réseau gracieuse, pas de crash
- [ ] Mode avion désactivé → reprise automatique

---

## C. Tests E2E automatisés web (Playwright)

À ajouter dans `attendee-ems-front/tests/e2e/` :

```ts
// tests/e2e/auth-refactor-v1.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Auth refactor v1', () => {
  test('root can access org without membership (bypass)', async ({ page }) => {
    await page.goto('/login');
    await page.fill('[name=email]', 'rgharghar@choyou.fr');
    await page.fill('[name=password]', 'Test1234!');
    await page.click('button[type=submit]');
    await page.waitForURL('**/dashboard');

    await page.click('[data-testid=org-switcher]');
    await page.click('text=Acme Corp');

    // Bypass root : accès aux organisations même sans membership
    await page.goto('/organizations');
    await expect(page.locator('text=Acme Corp')).toBeVisible();
  });

  test('agent only sees assigned events', async ({ page }) => {
    /* … */
  });

  test('switch org refreshes JWT and permissions', async ({ page }) => {
    /* … */
  });
});
```

Tests minimum à automatiser :
- [ ] Login OK / KO
- [ ] Switch org change l'org active
- [ ] Bypass root sur org sans membership
- [ ] Restriction agent (scope assigned)
- [ ] Logout efface le token

---

## D. Tests E2E automatisés mobile (Detox)

Si Detox est configuré dans le repo mobile, ajouter :

```ts
// e2e/authRefactor.test.ts
describe('Auth refactor v1 — mobile', () => {
  it('root can switch and bypass on Acme', async () => { /* … */ });
  it('session persists after app restart', async () => { /* … */ });
});
```

Si Detox n'est pas configuré : skip cette section, smoke manuel suffit pour cette PR.

---

## E. Tests API isolés (script curl)

Ajouter à `attendee-ems-back/scripts/test-palier3c-curl.sh` (ou étendre `test-palier2-curl.sh`) :

- [x] Login → 200 + JWT
- [x] `/auth/me/orgs` → liste correcte
- [x] `/auth/switch-org` → nouveau JWT
- [x] Bypass root sur `/organizations` → 200
- [x] Sans token → 401
- [x] `/auth/policy` sur Acme (root sans rôle) → 200
- [ ] **À ajouter** : login admin → permissions correctes dans le JWT
- [ ] **À ajouter** : login agent → scope `assigned` correctement appliqué sur `/events`

---

## Matrice de validation finale

| Scénario | Web manuel | Web Playwright | Mobile manuel | API curl |
|---|---|---|---|---|
| Login OK | ⬜ | ⬜ | ⬜ | ⬜ |
| Login KO | ⬜ | ⬜ | ⬜ | ⬜ |
| Switch org | ⬜ | ⬜ | ⬜ | ⬜ |
| Bypass root | ⬜ | ⬜ | N/A | ⬜ |
| Admin restriction | ⬜ | ⬜ | ⬜ | ⬜ |
| Agent scope assigned | ⬜ | ⬜ | ⬜ | ⬜ |
| Logout | ⬜ | ⬜ | ⬜ | N/A |
| Session restore (mobile) | N/A | N/A | ⬜ | N/A |

**Critère pour passer en pré-prod :** toutes les cases cochées, aucune régression vs prod actuelle.
