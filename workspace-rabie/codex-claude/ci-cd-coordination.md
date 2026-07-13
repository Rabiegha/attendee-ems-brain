# Coordination Codex / Claude — CI/CD LFD2026

> Date : 2026-07-12  
> Sujet : validation du chantier 0-CI / 0-MON avant passage a la suite  
> Objectif : garder une liste claire de ce que Rabie doit faire, ce que Claude doit coder/verifier, et ce que Codex doit expliquer/auditer.

---

## ⚡ MISE À JOUR 2026-07-13 — la partie « tests » de ce doc est désormais pilotée ailleurs

La section « Tests critiques a ajouter ou verifier » (§2) a été transformée en **chantier T dédié**,
avec des décisions qui **amendent la demande initiale à Claude** :

| Sujet | Décision 13/07 | Où c'est piloté |
| --- | --- | --- |
| Liste des tests event-critical | Chantier **T** créé : P0 (health, scan QR event, permissions) / P1 (PDF-badges, auth KO, BullMQ) / P2 — **liste à valider avant de coder** | [T-tests-event/README.md](../../workstreams/en-cours/lfd2026/T-tests-event/README.md) |
| `registration.e2e-spec.ts` + tests scan **session** + « user sans droit scanner » | **Déplacés au chantier H** (owner Corentin) — dépendent des règles métier H (privé/public, event vs session). **La V1 du CI/CD est livrée sans.** | [H · tests-sessions.md](../../workstreams/en-cours/lfd2026/H-inscriptions-session/tests-sessions.md) (tests H-T1→T14 + devs H-D1→D7) |
| Tests paiement/webhook | **Supprimés — pas de paiement pour cet event** | — |
| État réel du code scan (vérifié) | HMAC **strict et branché** sur `checkInByCode` ✅ (mais 0 E2E route) · déjà scanné = **409 `ALREADY_CHECKED_IN` + état serveur** (réconciliation offline mobile — reco : garder) · 🔴 **le re-scan n'est PAS loggé** (`AuditLog` Prisma jamais appelé) → dev à ajouter (H-D1) | [T §5bis](../../workstreams/en-cours/lfd2026/T-tests-event/README.md) |
| Charge des tests en CI | **Tout lancer à chaque PR** (échelle ~2-5 min, aucun souci) ; étagement/optimisations = post-event | [0-fondations/ci-cd-post-event.md](../../workstreams/en-cours/lfd2026/0-fondations/ci-cd-post-event.md) |

**Conséquence pour la demande à Claude (§2)** : garder T1 `health`, T2 `scan-qr` (scan principal
event uniquement, contrat 409 à figer, cas « token HMAC falsifié »), T3 `permissions-event` ;
**retirer** `registration.e2e-spec.ts` (→ H) et tout ce qui touche paiement.

Le reste de ce doc (secrets, PAT GHCR, clé SSH, branch protection, scripts legacy) reste valable tel quel.

---

## ⚡ MISE À JOUR 2026-07-13 (soir) — pipeline VALIDÉ de bout en bout (staging back)

Exécution pilotée par la checklist [finalisation-ci-cd-et-livraison.md](../finalisation-ci-cd-et-livraison.md) :

| Sujet | État |
| --- | --- |
| Prérequis ops (§ « ce que Rabie doit faire ») | ✅ TOUS faits : clé SSH GHA, secrets `VPS_HOST`/`VPS_USER`/`VPS_SSH_KEY` (back+front), PAT GHCR loggé root+debian sur le VPS |
| CI back | ✅ **Verte 18/18 e2e** après 3 fixes : `REDIS_HOST` manquant (job pendu 3h49) · seed e2e inexistant (`test/seed-e2e.ts` créé — les specs supposaient des users jamais seedés) · throttler 429 + assertions cookie obsolètes |
| CD staging back | ✅ **Validé en conditions réelles** : run `29266715214` → `ems-staging-api` sur image GHCR `28eb5d2…`, `/api/health` full ok, version = sha |
| Workflows sur `main` | ✅ Découverte : `workflow_run`/`workflow_dispatch` exigent le workflow sur la **branche par défaut** → merge `staging → main` (`4362745`). Rien ne se déploie en auto (CD prod = dispatch + confirm) |
| CD staging front | ✅ Créé (`bca6eca`) + 🔴 bug swap prod corrigé (`dist/staging` survivait pas au deploy prod). ⚠️ Même règle : devra être sur `main` du repo front pour être déclenchable |
| CD prod back / front | ⏸ **EN ATTENTE** (fenêtre 22h-06h ou hotfix). ⚠️ `main` = tout staging désormais : plus un no-op |
| Vhost staging | 🔴→✅ Incident découvert à l'audit : vhost **disparu du VPS** (404 ~3 j, commit orphelin balayé) — restauré + commité sur dev/staging/main |

**Point de coordination humain** : ✅ point fait avec Corentin (13/07) — explication de la nouvelle
méthode de travail CI/CD, revue de ce qui était fait, répartition des tâches de la semaine.

---

## 1. Contexte rapide

Claude a code une premiere version du chantier CI/CD sur les branches `chore/ci-cd` :

- back : CI, healthcheck enrichi, deploy depuis GHCR, CD staging, CD prod, monitoring.
- front : CI, CD prod front, lint repare, smoke test autonome.

Rabie est en phase de comprehension/validation avant merge et activation reelle.

Le point important : le code CI/CD existe, mais certains pre-requis operationnels doivent etre faits hors repo :

- secrets GitHub Actions,
- acces SSH GitHub Actions vers VPS,
- acces Docker du VPS vers GHCR,
- protection des branches,
- validation des tests critiques avant event.

---

## 2. A demander a Claude

### Tests critiques a ajouter ou verifier

Pas besoin de couvrir tout le produit avant l'event. Il faut couvrir les chemins qui casseraient l'exploitation.

Priorite tests a adapter au code existant :

1. Tests healthcheck :
   - `/health` retourne `status=ok` si db/redis/migrations OK.
   - `/health` retourne HTTP 503 si db KO.
   - `/health` expose bien `version`.

2. Tests auth critique :
   - login OK.
   - refresh token OK.
   - token invalide/refuse.
   - session expiree/refusee proprement.

3. Tests scan QR / validation billet :
   - QR valide accepte.
   - QR invalide refuse.
   - QR deja scanne refuse ou signale selon regle metier.
   - controle HMAC bien applique sur les chemins de scan critiques.

4. Tests inscription / paiement si concerne :
   - creation inscription nominale.
   - statut paiement attendu.
   - cas erreur paiement/webhook si deja branche.

5. Tests permissions admin critiques :
   - admin autorise sur actions sensibles.
   - utilisateur non admin refuse.
   - controle tenant/event si multi-tenant actif.

### Verdict Codex sur les E2E actuels

Question a traiter :

```txt
Les tests E2E actuels suffisent-ils pour les chemins event critiques ?
```

Verdict Codex :

```txt
Non, pas encore.
```

Les tests existants sont utiles pour la CI, mais ils couvrent surtout :

- auth basique,
- refresh token/logout,
- acces `/users`,
- acces `/organizations/me`,
- creation utilisateur,
- smoke front login,
- QR token en unitaire.

Ils ne couvrent pas encore assez les chemins event critiques :

- scan QR / validation billet en vrai E2E API,
- billet deja scanne,
- billet inconnu/invalide,
- token QR modifie,
- droits scanner/admin/non-autorise,
- inscription/registration event,
- healthcheck reel avec Postgres/Redis dans un E2E.

Demande prioritaire a Claude :

```txt
Ajoute les E2E minimaux event-critical avant validation du chantier CI/CD :

1. health.e2e-spec.ts
   - GET /health retourne status=ok avec Postgres/Redis/migrations OK.
   - verifie presence version.

2. scan-qr.e2e-spec.ts
   - QR valide accepte.
   - QR invalide/token modifie refuse.
   - billet deja scanne refuse ou signale selon regle metier.
   - utilisateur sans droit scanner refuse.

3. permissions-event.e2e-spec.ts
   - admin/staff/scanner autorises selon role.
   - utilisateur non autorise refuse.
   - verifier tenant/event si applicable.

4. registration.e2e-spec.ts si faisable avant event
   - creation inscription nominale.
   - statut attendu.
   - presence/generation QR ou badge si lie au flow actuel.
```

Objectif : ne pas chercher 100% de couverture, seulement proteger les parcours qui casseraient l'exploitation pendant l'event.

### Organisation scripts deploy

Demander a Claude si on garde `scripts/deploy-from-registry.sh` tel quel ou si on le range dans un dossier plus explicite, par exemple :

```txt
scripts/deploy/deploy-from-registry.sh
```

Si deplacement :

- mettre a jour `cd-prod.yml`,
- mettre a jour `cd-staging.yml`,
- verifier les chemins dans les docs,
- ne pas casser les commandes executees sur le VPS.

### Incoherence scripts front legacy vs nginx

Point a faire verifier/corriger par Claude :

```txt
Confirme que nginx sert bien /opt/ems-attendee/frontend/dist en prod.
Si oui, nettoie ou corrige les copies vers /opt/ems-attendee/backend/frontend dans les scripts legacy.
```

Constat actuel :

- `docker-compose.prod.yml` monte le front dans nginx via :

  ```txt
  ../frontend/dist:/usr/share/nginx/html:ro
  ```

- donc, si le compose est lance depuis `/opt/ems-attendee/backend`, nginx sert :

  ```txt
  /opt/ems-attendee/frontend/dist
  ```

- mais `deploy-front.sh` et `deploy.sh` copient aussi le build vers :

  ```txt
  /opt/ems-attendee/backend/frontend/
  ```

Probleme :

```txt
Si nginx ne lit plus /opt/ems-attendee/backend/frontend/,
alors cette copie est probablement une ancienne etape morte.
```

Origine probable :

```txt
Le dossier /opt/ems-attendee/backend/frontend a probablement ete cree par les anciens scripts
deploy-front.sh / deploy.sh, qui buildaient le front sur le VPS puis copiaient aussi le dist dans
le repo backend.
```

Important : l'ancien deploy front pouvait quand meme marcher, non pas grace a
`/opt/ems-attendee/backend/frontend`, mais parce que `npm run build` mettait deja a jour :

```txt
/opt/ems-attendee/frontend/dist
```

qui est justement le dossier servi par nginx.

Ce qu'il faut demander a Claude :

1. Verifier le montage nginx reel en prod.
2. Verifier si `/opt/ems-attendee/backend/frontend/` est encore utilise par une conf nginx ancienne ou un autre service.
3. Si non utilise :
   - supprimer les copies inutiles dans `deploy-front.sh`, `deploy.sh`, `deploy-logs.sh`, `deploy-back-logs.sh` ;
   - garder uniquement `/opt/ems-attendee/frontend/dist` comme source servie par nginx ;
   - verifier que le nouveau `cd-prod.yml` front deploye bien dans ce dossier.
4. Si encore utilise :
   - documenter pourquoi il y a deux destinations ;
   - eviter que les deux dossiers divergent.

Procedure safe cote VPS si le dossier semble mort :

```bash
cd /opt/ems-attendee/backend
grep -R "backend/frontend\\|/frontend\\|/usr/share/nginx/html" nginx docker-compose*.yml
docker inspect ems-nginx | grep -A5 -B5 frontend
```

Si tout confirme que nginx ne sert pas `/opt/ems-attendee/backend/frontend`, ne pas supprimer
directement. Renommer d'abord :

```bash
mv /opt/ems-attendee/backend/frontend /opt/ems-attendee/backend/frontend.legacy-2026-07-12
docker compose -f /opt/ems-attendee/backend/docker-compose.prod.yml restart nginx
curl -I https://attendee.fr
```

Si le front public reste OK apres quelques jours, supprimer le dossier legacy.

Risque :

```txt
Un deploy front peut sembler reussir mais copier au mauvais endroit,
donc nginx continue a servir l'ancien front.
```

### Scripts legacy : fallback manuel ou suppression ?

Point de clarification a demander a Claude :

```txt
Dans le nouveau systeme GHA, deploy-back.sh et deploy-front.sh ne sont plus le chemin principal.
Est-ce qu'on les garde comme fallback manuel pendant l'event, ou est-ce qu'on les archive/supprime ?
```

Chemin principal vise :

```txt
Back prod :
GitHub Actions cd-prod.yml
-> SSH VPS
-> scripts/deploy-from-registry.sh prod <sha>

Front prod :
GitHub Actions cd-prod.yml front
-> build dist sur GHA
-> upload dist.tar.gz
-> swap /opt/ems-attendee/frontend/dist
```

Donc :

```txt
deploy-back.sh  = legacy si le CD back GHA est valide
deploy-front.sh = legacy si le CD front GHA est valide
```

Mais si on les garde, ce doit etre en tant que :

```txt
fallback manuel temporaire
```

Recommandation Codex pour l'event :

```txt
GHA = chemin principal
scripts legacy corriges = fallback manuel temporaire
suppression/archive plus tard, une fois CD staging/prod rode
```

Pourquoi les corriger quand meme ?

1. Eviter deux chemins contradictoires :
   - nouveau front CD -> `/opt/ems-attendee/frontend/dist`
   - ancien script front -> copie aussi vers `/opt/ems-attendee/backend/frontend`

2. Garder un vrai plan B :
   - si GitHub Actions, GHCR ou les secrets cassent pendant l'event, on peut avoir besoin d'un deploy manuel.

3. Eviter le piege `sudo bash` :
   - `git pull`/`git fetch` doivent tourner avec l'utilisateur qui a la config SSH GitHub, probablement `debian`.
   - `sudo` seulement pour Docker/system si necessaire.

Dans le nouveau systeme GHA, verifier aussi que `VPS_USER` peut vraiment executer :

```txt
git fetch / git checkout
docker pull
docker compose up
lire .env.production
```

Si `VPS_USER=debian` ne peut pas lancer Docker directement, choisir une strategie claire :

```txt
Option A : ajouter debian au groupe docker
Option B : adapter les scripts pour utiliser sudo docker / sudo docker compose
```

Attention : ne pas lancer tout le script en `sudo bash`, sinon on peut reperdre la config SSH/Git
de `debian` et retomber sur :

```txt
Permission denied (publickey)
```

### Cle SSH dediee GitHub Actions

Demande possible a Claude :

```txt
Cree une cle SSH ed25519 dediee GitHub Actions pour le deploiement, ajoute la cle publique a l'utilisateur debian sur le VPS, puis dis-moi exactement quoi mettre dans GitHub Secret VPS_SSH_KEY sans l'ecrire dans le repo.
```

Attention :

- ne jamais committer la cle privee,
- ne pas la stocker dans un fichier du repo,
- la coller uniquement dans GitHub Secrets.

---

## 3. Ce que Claude attend de Rabie

### Secrets GitHub Actions a creer

Dans chaque repo concerne si le workflow en a besoin :

```txt
GitHub -> repo -> Settings -> Secrets and variables -> Actions -> New repository secret
```

Secrets back attendus :

```txt
VPS_HOST        = IP ou domaine du VPS
VPS_USER        = utilisateur Linux de deploy, probablement debian
VPS_SSH_KEY     = cle privee SSH complete dediee GitHub Actions
NOTIFY_WEBHOOK  = webhook notification, optionnel
```

Important :

- `VPS_USER=debian` n'est pas un mot de passe.
- Le mot de passe `debian` ne doit pas etre mis dans GitHub.
- La connexion se fait par cle SSH : `ssh -i VPS_SSH_KEY debian@VPS_HOST`.

### Acces GHCR cote VPS

Le VPS doit pouvoir pull l'image Docker privee :

```bash
docker pull ghcr.io/rabiegha/ems-api:<sha>
```

Il faut donc, sur le VPS, faire une fois :

```bash
docker login ghcr.io -u Rabiegha
```

Puis coller un PAT GitHub avec permission :

```txt
read:packages
```

Ce PAT sert au VPS pour lire les images GHCR. Il ne sert pas a GitHub Actions pour pousser l'image.

---

## 4. A demander a Codex

### Protection des branches

Demander a Codex :

```txt
Fais-moi une liste des protections a activer sur chaque branche main et staging, avec un petit guide.
```

Attendu :

#### `main`

- Require pull request before merging : oui.
- Require approvals : oui, 1 approbation.
- Require status checks : oui, CI obligatoire.
- Require branches to be up to date before merging : recommande si pas trop bloquant.
- Block force pushes : oui.
- Block deletions : oui.
- Require linear history : recommande.
- Restrict who can push : optionnel selon equipe.

#### `staging`

- Require pull request before merging : oui.
- Require status checks : oui, CI obligatoire.
- Require approvals : optionnel, mais recommande si event proche.
- Block force pushes : oui.
- Block deletions : oui.
- Require linear history : optionnel.

Guide GitHub :

```txt
Repo GitHub
-> Settings
-> Branches
-> Branch protection rules
-> Add branch ruleset / Add rule
-> Branch name pattern: main ou staging
```

### Audit tests E2E actuels

Demander a Codex :

```txt
C'est quoi la liste des tests E2E actuels back/front, ou vivent-ils, et est-ce suffisant pour l'event ?
```

Reponse attendue :

- lister les fichiers `test/*.e2e-spec.ts` du back,
- lister les fichiers `tests/e2e/*.spec.ts` du front,
- dire ce qui est deja couvert,
- dire les trous critiques avant event,
- proposer un plan court de tests a ajouter.

---

## 5. A faire par Rabie

### Protection de `main` et `staging`

Action :

```txt
Repo GitHub -> Settings -> Branches -> Branch protection rules
```

Activer au minimum :

- PR obligatoire vers `main`.
- PR obligatoire vers `staging`.
- CI obligatoire avant merge.
- 1 approbation obligatoire sur `main`.
- force-push interdit.
- deletion interdite.

### Creation du PAT GHCR

Objectif : permettre au VPS de pull les images privees GHCR.

Guide detaille :

1. Aller sur GitHub.

2. Cliquer sur la photo de profil en haut a droite.

3. Aller dans :

   ```txt
   Settings
   ```

4. Dans le menu gauche, tout en bas, aller dans :

   ```txt
   Developer settings
   ```

5. Aller dans :

   ```txt
   Personal access tokens
   -> Tokens (classic)
   ```

6. Cliquer sur :

   ```txt
   Generate new token
   -> Generate new token (classic)
   ```

7. Remplir le nom :

   ```txt
   ems-vps-ghcr-read
   ```

8. Choisir une expiration :

   ```txt
   90 days
   ```

   ou :

   ```txt
   1 year
   ```

   Conseil : `1 year` est plus confortable pour la prod, mais mettre un rappel calendrier avant expiration.

9. Cocher uniquement le scope :

   ```txt
   read:packages
   ```

   Ne pas cocher `write:packages` pour le VPS : il doit seulement pull, pas push.

10. Cliquer sur :

    ```txt
    Generate token
    ```

11. Copier le token immediatement.

    GitHub ne l'affichera plus apres fermeture de la page.

12. Se connecter au VPS en SSH avec l'utilisateur de deploy, probablement `debian`.

13. Sur le VPS, lancer :

    ```bash
    docker login ghcr.io -u Rabiegha
    ```

14. Quand Docker demande :

    ```txt
    Password:
    ```

    coller le PAT GitHub.

15. Docker doit repondre quelque chose comme :

    ```txt
    Login Succeeded
    ```

16. Verifier ensuite que Docker a garde l'auth cote VPS :

    ```bash
    ls -la ~/.docker/config.json
    ```

17. Quand une premiere image GHCR existera, tester :

    ```bash
    docker pull ghcr.io/rabiegha/ems-api:<sha>
    ```

Note :

- si aucune image GHCR n'a encore ete publiee, `docker pull` peut echouer car le package n'existe pas encore.
- apres premier push CI sur `staging` ou `main`, verifier GitHub -> Rabiegha -> Packages.
- ne pas coller le PAT dans un chat ou dans un fichier du repo.
- si le PAT expire, les deploys echoueront probablement sur `docker pull` avec `unauthorized`.

### Secrets GitHub Actions

Creer les secrets suivants dans le repo back :

```txt
VPS_HOST
VPS_USER
VPS_SSH_KEY
NOTIFY_WEBHOOK
```

Eventuellement creer les memes dans le repo front si son CD prod les utilise aussi.

### Ne pas partager les secrets dans les chats

Eviter de donner en clair :

- PAT GitHub,
- cle privee SSH,
- mot de passe VPS,
- contenu `.env.production`.

Dire plutot a Claude/Codex :

```txt
C'est configure.
Voici le nom du secret, pas sa valeur.
```

---

## 6. Schema mental CI/CD

```txt
CI GitHub Actions
  -> lint / typecheck / tests
  -> build image Docker back
  -> push image dans GHCR avec tag <sha>

CD GitHub Actions
  -> verifie que la CI est verte pour le meme SHA
  -> se connecte au VPS en SSH
  -> demande au VPS de pull ghcr.io/rabiegha/ems-api:<sha>
  -> docker compose up -d --no-build
  -> migrations Prisma
  -> healthcheck
  -> rollback si KO
```

Les trois acces a ne pas melanger :

```txt
CI -> GHCR push
  Auth : GITHUB_TOKEN automatique GitHub Actions

CD -> VPS ssh
  Auth : VPS_HOST / VPS_USER / VPS_SSH_KEY

VPS -> GHCR pull
  Auth : docker login ghcr.io avec PAT read:packages
```

---

## 7. Questions ouvertes avant validation

- Les secrets GitHub Actions existent-ils deja sur `attendee-ems-back` ?
- Le VPS a-t-il deja un `docker login ghcr.io` valide ?
- Le package GHCR `ems-api` existe-t-il apres premier push sur `staging` ou `main` ?
- Les URLs healthcheck sont-elles coherentes entre code, nginx et scripts ?
  - code Nest : `/health`
  - public/proxy possible : `/api/health`
- Les tests E2E actuels suffisent-ils pour les chemins event critiques ?
- Faut-il ajouter un scan migrations risquées en CI PR, en plus du verrou CD prod ?
