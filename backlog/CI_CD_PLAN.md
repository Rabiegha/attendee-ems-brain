# Plan CI/CD — Tests automatises et deploiements securises

## Contexte et objectifs

La plateforme Attendee est desormais utilisee en production. Ce plan repond a deux problematiques :

1. **Tester automatiquement le code avant chaque deploiement** — eviter les regressions.
2. **Deployer uniquement en heures creuses** (sauf urgence explicite) — reduire l'impact sur les utilisateurs.

## Comment les tests fonctionnent concrètement

### Les trois niveaux de tests

**Niveau 1 — Tests unitaires (logique pure)**
Testent une fonction isolement, sans navigateur ni base de donnees. Utiles pour detecter les bugs de calcul ou de transformation de donnees. Rapides (quelques secondes). Ne detectent pas les problemes visuels.

**Niveau 2 — Tests E2E backend (API comme un client reel)**
Envoient de vraies requetes HTTP a l'API avec une vraie base de donnees de test. C'est l'equivalent d'un humain qui clique "S'inscrire" — mais au niveau reseau. Exemple :

```typescript
// Ce test reproduit exactement ce que fait le frontend a l'inscription
await request(app)
  .post(`/public/events/${publicToken}/register`)
  .send({ firstName: 'Jean', email: 'jean@test.fr' })
  .expect(201)

// Et verifie que l'email de confirmation a bien ete envoye
expect(emailService.send).toHaveBeenCalledWith(...)
```

Si une modification casse l'envoi d'email a l'inscription, ce test echoue et le deploy est bloque.

**Niveau 3 — Tests Playwright (vrai navigateur pilote automatiquement)**
Playwright ouvre Chrome, clique sur les boutons, remplit les formulaires, verifie ce qui est visible. C'est exactement comme un humain qui utilise l'application :

```typescript
await page.getByRole('button', { name: 'Creer un evenement' }).click()
await page.getByLabel('Nom').fill('Conference 2026')
await page.getByRole('button', { name: 'Sauvegarder' }).click()
await expect(page.getByText('Conference 2026')).toBeVisible()
```

Si un bouton n'est pas visible ou si un formulaire ne soumet pas, le test echoue.

### Comment couvrir des centaines de fonctionnalites sans tout ecrire

Les tests s'ecrivent une seule fois, puis la machine les rejoue a chaque deploy :

```
Aujourd'hui   : on ecrit un test "l'email est envoye a l'inscription"
Dans 3 mois   : une modif touche le systeme de notifications
CI             : rejoue automatiquement tous les tests existants
                 -> le test "email inscription" echoue
                 -> deploy bloque
                 -> on sait exactement ce qui est casse avant la prod
```

La couverture se construit progressivement :

1. **D'abord les parcours critiques** — les actions qui, si elles cassent, bloquent tous les utilisateurs (login, inscription, check-in, envoi de badge).
2. **A chaque bug trouve en prod** — on ecrit un test qui reproduit ce bug pour qu'il ne revienne jamais.
3. **Avec le temps** — le filet de securite couvre de plus en plus de fonctionnalites.

### Ce que ce plan protege en priorite

| Test | Scenario protege |
|---|---|
| `auth.e2e-spec.ts` | Personne ne peut se connecter |
| `public-registration.e2e-spec.ts` | Les attendees ne peuvent plus s'inscrire |
| `checkin.e2e-spec.ts` | Le check-in le jour J ne fonctionne plus |
| `events.e2e-spec.ts` | On ne peut plus creer ou gerer des evenements |
| Playwright `login.spec.ts` | L'interface de connexion est cassee visuellement |
| Playwright `event-create.spec.ts` | Le formulaire de creation d'event est casse |
| Playwright `public-register.spec.ts` | Le formulaire d'inscription public est casse |

Ces tests couvrent les regressions qui rendraient l'application inutilisable. Les fonctionnalites secondaires se couvrent progressivement au fil des sprints.

---

## Architecture de la solution

```
feature/* ──► dev ──────────────────────────────► main (prod)
                                                      │
                                              workflow_dispatch
                                     ┌────────────────┴────────────────┐
                              Deploy normal                      Deploy hotfix
                              22h-06h Paris                      Toute heure
                              confirm=DEPLOY                     hotfix=true
                              + healthcheck                      confirm=DEPLOY
                              + auto-rollback                    + notification
```

**Repos couverts** : `attendee-ems-back`, `attendee-ems-front`
**Hors scope** : `attendee-ems-mobile` (Expo/EAS), `attendee-ems-print-client` (Electron)

---

## Decisions actees

| Sujet | Decision |
|---|---|
| Plateforme CI/CD | GitHub Actions |
| Branche d'integration | `dev` (existante) — 100% du travail quotidien |
| Branche de production | `main` (protegee) |
| Trigger PR dev -> main | CI verte obligatoire + **1 reviewer requis** |
| Deploy normal | `workflow_dispatch`, creneau 22h-06h Europe/Paris |
| Deploy hotfix | `workflow_dispatch` avec `hotfix=true` + `confirm=DEPLOY`, toute heure + notification systematique |
| Niveau de tests | Couverture moyenne (lint + typecheck + unit + E2E critiques + Playwright) |
| Migrations risquees | Check `DROP`/`ALTER COLUMN` — exige `migration_risky=true` |
| Rollback | Healthcheck post-deploy + auto-rollback (git SHA + image Docker taguee) |
| Staging | Non pour l'instant |

---

## Phase 1 — Strategie de branches et protection de main

### 1.1 Flux de travail quotidien

```bash
# Travail au quotidien
git checkout dev && git pull origin dev
git checkout -b feature/ma-fonctionnalite
# ... commits ...
# Ouvrir PR : feature/* -> dev  (CI rapide)

# Release vers prod
# Ouvrir PR : dev -> main  (CI complete + 1 reviewer)
# Merge = code pret a deployer
# Lancer deploy.yml manuellement la nuit
```

### 1.2 Branch protection sur main (a configurer dans GitHub)

**Settings > Branches > Add rule > `main`** :

- Require a pull request before merging
- Require approvals : **1**
- Require status checks : `lint-typecheck`, `unit-tests`, `e2e-tests`, `build`
- Require branches to be up to date before merging
- Do not allow bypassing the above settings

---

## Phase 2 — CI : tests automatises sur chaque PR et push

### 2.1 Workflow CI Backend

**Fichier** : `attendee-ems-back/.github/workflows/ci.yml`
**Declenchement** : `push` toutes branches + `pull_request` vers `dev` ou `main`

| Job | Commandes | Notes |
|---|---|---|
| `lint-typecheck` | `npm ci`, `npm run lint`, `tsc --noEmit` | Parallele |
| `unit-tests` | `npm ci`, `npm test` | Parallele |
| `build` | `npm ci`, `npm run build` | Parallele |
| `e2e-tests` | Services Postgres + Redis GHA, `prisma migrate deploy`, `npm run test:e2e` | Depend de `lint-typecheck` |

**Services GHA pour e2e** : `postgres:16-alpine` + `redis:7-alpine` avec variables injectees depuis les secrets. Reutilise `test/jest-e2e.json` et `test/setup-e2e.ts` existants.

### 2.2 Workflow CI Frontend

**Fichier** : `attendee-ems-front/.github/workflows/ci.yml`
**Declenchement** : `push` toutes branches + `pull_request` vers `dev` ou `main`

| Job | Commandes | Notes |
|---|---|---|
| `lint-typecheck` | `npm ci`, `npm run lint`, `npm run typecheck` | Parallele |
| `unit-tests` | `npm ci`, `npm test -- --run` | Vitest non-interactif |
| `build` | `npm ci`, `VITE_API_BASE_URL=... npm run build` | Parallele |
| `e2e-smoke` | `playwright install --with-deps chromium`, `playwright test` | `playwright.config.ts` detecte `CI=true` |

### 2.3 Tests E2E a ecrire

**Backend** (`test/*.e2e-spec.ts`) — reutilisent le pattern de `test/app.e2e-spec.ts` :

| Fichier | Cas couverts |
|---|---|
| `test/auth.e2e-spec.ts` | Login, refresh token, token expire, mauvais mdp |
| `test/events.e2e-spec.ts` | Creation event + EventSetting auto, list par org, RBAC |
| `test/public-registration.e2e-spec.ts` | `POST /public/events/:token/register`, duplicate, capacite |
| `test/checkin.e2e-spec.ts` | Check-in, double check-in bloque, check-out |

**Frontend** (Playwright, `tests/e2e/*.spec.ts`) :

| Fichier | Cas couverts |
|---|---|
| `tests/e2e/event-create.spec.ts` | Creation event minimal, verification dans la liste |
| `tests/e2e/public-register.spec.ts` | Soumission formulaire d'inscription public |

---

## Phase 3 — Deploiement controle avec garde horaire et rollback

### 3.1 Action composite check-deploy-window

**Fichier** : `.github/actions/check-deploy-window/action.yml` (dans chaque repo)

Logique :

1. Lire l'heure courante en `Europe/Paris`
2. Si heure dans `[22:00, 06:00]` → continuer
3. Si `hotfix=true` → continuer + emission d'une notification d'alerte
4. Sinon → `exit 1` : `"Deploy rejete : hors creneau 22h-06h. Utiliser hotfix=true pour une urgence."`

### 3.2 Check migration risquee

Dans le job `deploy` backend, avant `prisma migrate deploy` :

1. Identifier les nouveaux fichiers `prisma/migrations/**/*.sql` (commits depuis `.last_good_sha`)
2. Grep `DROP TABLE`, `DROP COLUMN`, `ALTER COLUMN`, `TRUNCATE`
3. Si detecte ET `migration_risky != 'true'` → `exit 1` avec message explicatif
4. Si `migration_risky=true` → continuer avec log d'avertissement

### 3.3 Workflow Deploy Backend

**Fichier** : `attendee-ems-back/.github/workflows/deploy.yml`
**Declenchement** : `workflow_dispatch` uniquement

**Inputs** :

| Input | Type | Requis | Description |
|---|---|---|---|
| `confirm` | string | oui | Doit valoir exactement `DEPLOY` |
| `hotfix` | boolean | non | `true` = bypass time-window |
| `migration_risky` | boolean | non | `true` = confirmer migration destructive |

**Sequence de jobs** :

```
guard -> verify-ci -> risky-migration-check -> deploy -> healthcheck
                                                              │
                                                        (si echec)
                                                          rollback -> notify
```

| Job | Detail |
|---|---|
| `guard` | `confirm == 'DEPLOY'` + action `check-deploy-window` |
| `verify-ci` | `gh api .../commits/main/check-runs` → refuse si CI pas verte sur HEAD de main |
| `risky-migration-check` | Scan SQL des nouvelles migrations (cf. 3.2) |
| `deploy` | SSH VPS (`appleboy/ssh-action@v1`), sauvegarde SHA dans `.last_good_sha`, `docker tag ems-api:latest ems-api:rollback`, `./deploy-back.sh` |
| `healthcheck` | Poll `https://api.attendee.fr/health` 30x5s ; verifie champs `db`, `redis`, `migrations` dans le JSON |
| `rollback` | `if: failure()` ; SSH : `git reset --hard $(cat .last_good_sha)`, `docker tag ems-api:rollback ems-api:latest`, `docker compose up -d api` |
| `notify` | Toujours execute ; succes ou echec + lien run GHA |

**Secrets a configurer dans GitHub (Settings > Secrets and variables > Actions)** :

| Secret | Valeur |
|---|---|
| `VPS_SSH_KEY` | Cle SSH privee ED25519 |
| `VPS_HOST` | `51.75.252.74` |
| `VPS_USER` | `debian` |
| `NOTIFY_WEBHOOK` | Webhook Slack ou URL API email |

### 3.4 Workflow Deploy Frontend

**Fichier** : `attendee-ems-front/.github/workflows/deploy.yml`
Meme structure que le backend, sans `risky-migration-check`.

Differences :

| Aspect | Detail |
|---|---|
| Snapshot rollback | SSH : `cp -r frontend/ frontend.previous/` avant overwrite |
| Healthcheck | `curl https://attendee.fr` retourne 200 + hash du nouveau bundle present dans `index.html` |
| Rollback | SSH : `rm -rf frontend/`, `mv frontend.previous/ frontend/`, restart nginx |

### 3.5 Modifications des scripts de deploy existants

**`attendee-ems-back/deploy-back.sh`** :

1. Avant `git pull` : `git rev-parse HEAD > .last_good_sha`
2. Avant `docker compose up --build` : `docker tag ems-api:latest ems-api:rollback 2>/dev/null || true`
3. Apres `prisma migrate deploy` : verifier exit code explicitement
4. Ajouter en fin : `npx prisma migrate status` — echoue dur si migrations inconsistantes

**`attendee-ems-front/deploy-front.sh`** :

1. Avant `cp -r dist/* ...` : `cp -r $DEPLOY_DIR/backend/frontend $DEPLOY_DIR/backend/frontend.previous 2>/dev/null || true`

### 3.6 Enrichir l'endpoint /health

L'endpoint `GET /health` doit retourner :

```json
{
  "status": "ok",
  "db": "ok",
  "redis": "ok",
  "migrations": "ok",
  "version": "abc1234"
}
```

Retourner HTTP 503 si un check echoue — permet au healthcheck GHA de detecter l'echec automatiquement.

---

## Phase 4 — Notifications

Evenements notifies via `NOTIFY_WEBHOOK` (Slack ou email) :

| Evenement | Contenu |
|---|---|
| Deploy demarre | Repo, branch, SHA, auteur, `hotfix=true/false` |
| Deploy succes | Idem + duree + lien run GHA |
| Healthcheck KO — rollback en cours | Alerte prioritaire |
| Rollback termine | SHA restaure, version en prod |
| Migration risquee deployee | Noms des fichiers SQL concernes |

---

## Recapitulatif des fichiers

### A creer

| Fichier | Description |
|---|---|
| `attendee-ems-back/.github/workflows/ci.yml` | Pipeline CI backend |
| `attendee-ems-back/.github/workflows/deploy.yml` | Deploy backend avec garde horaire |
| `attendee-ems-back/.github/actions/check-deploy-window/action.yml` | Verification creneau horaire |
| `attendee-ems-front/.github/workflows/ci.yml` | Pipeline CI frontend |
| `attendee-ems-front/.github/workflows/deploy.yml` | Deploy frontend |
| `attendee-ems-front/.github/actions/check-deploy-window/action.yml` | Verification creneau horaire |
| `attendee-ems-back/test/auth.e2e-spec.ts` | Tests E2E auth |
| `attendee-ems-back/test/events.e2e-spec.ts` | Tests E2E events |
| `attendee-ems-back/test/public-registration.e2e-spec.ts` | Tests E2E registration publique |
| `attendee-ems-back/test/checkin.e2e-spec.ts` | Tests E2E check-in |
| `attendee-ems-front/tests/e2e/event-create.spec.ts` | Playwright creation event |
| `attendee-ems-front/tests/e2e/public-register.spec.ts` | Playwright registration publique |

### A modifier

| Fichier | Modification |
|---|---|
| `attendee-ems-back/deploy-back.sh` | Sauvegarde SHA, tag image rollback, migrate status strict |
| `attendee-ems-front/deploy-front.sh` | Snapshot `frontend.previous/` avant overwrite |
| Health controller backend | Enrichir : DB, Redis, migrations, version |

### Configuration GitHub (hors code)

| Action | Ou |
|---|---|
| Branch protection `main` (back + front) | Settings > Branches |
| Secrets : `VPS_SSH_KEY`, `VPS_HOST`, `VPS_USER`, `NOTIFY_WEBHOOK` | Settings > Secrets and variables > Actions |
| Passer `dev` comme branche par defaut | Settings > Default branch |

---

## Plan de verification

| # | Test | Resultat attendu |
|---|---|---|
| 1 | PR `feature/test-ci` -> `dev` dans chaque repo | 4 jobs CI passent |
| 2 | Commit TypeScript casse dans une PR | Merge bloque |
| 3 | Lancer `deploy.yml` a 14h sans `hotfix` | Job `guard` echoue "hors creneau 22h-06h" |
| 4 | Lancer `deploy.yml` a 23h avec `confirm=DEPLOY` | Sequence complete : guard + verify-ci + deploy + healthcheck OK |
| 5 | Lancer `deploy.yml` a 14h avec `hotfix=true` + `confirm=DEPLOY` | Deploy execute + notification alerte hotfix envoyee |
| 6 | Migration avec `DROP TABLE`, sans `migration_risky=true` | Job `risky-migration-check` echoue |
| 7 | Meme migration avec `migration_risky=true` | Deploy + log d'avertissement dans le resume |
| 8 | Commit cassant `/health`, deploy de nuit | Rollback auto, site reste up, notification envoyee |
| 9 | `curl https://api.attendee.fr/health` | JSON `{status: ok, db: ok, redis: ok, migrations: ok}` |
| 10 | PR `dev` -> `main` sans reviewer | Merge bloque (branch protection) |
| 11 | PR `dev` -> `main` avec CI rouge + reviewer | Merge bloque meme avec reviewer |

---

## Ordre d'implementation

1. **Enrichir `/health`** — prerequis du healthcheck, valeur immediate
2. **Modifier `deploy-back.sh` et `deploy-front.sh`** — prerequis du rollback automatique
3. **Creer les workflows CI** (back + front) — valeur immediate sur les PRs
4. **Configurer branch protection sur `main`** — apres que la CI tourne
5. **Ecrire les tests E2E supplementaires** — en parallele avec les etapes suivantes
6. **Creer les workflows de deploy** (back + front)
7. **Creer l'action composite `check-deploy-window`**
8. **Configurer les secrets GitHub**
9. **Valider le plan de verification complet**
