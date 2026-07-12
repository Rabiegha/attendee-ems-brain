# CI/CD — GitHub Actions, GHCR, VPS, secrets et tests event-critical

> **Date :** 2026-07-12 · **Contexte :** validation du chantier 0-CI / 0-MON LFD2026.
> Comprendre ce que Claude a codé avant d'activer le pipeline réel.
> **Réfs :** [0-CI-plan.md](../workstreams/en-cours/lfd2026/0-fondations/0-CI-plan.md) ·
> [pont Codex/Claude](../workspace-rabie/codex-claude/ci-cd-coordination.md).

---

## 1. Schéma mental général

Le pipeline a deux grandes parties :

```txt
CI GitHub Actions
  -> lint / typecheck / tests
  -> build image Docker back
  -> push image dans GHCR avec tag <sha>

CD GitHub Actions
  -> vérifie que la CI est verte pour le même SHA
  -> se connecte au VPS en SSH
  -> demande au VPS de pull ghcr.io/rabiegha/ems-api:<sha>
  -> docker compose up -d --no-build
  -> migrations Prisma
  -> healthcheck
  -> rollback si KO
```

Phrase clé :

```txt
GitHub Actions pilote le déploiement.
Le VPS exécute le déploiement.
GHCR fournit l'image Docker.
```

Le VPS ne décide pas seul de pull une image. Le CD GitHub Actions se connecte au VPS et lui donne
l'ordre de faire le `docker pull`, le `docker compose up`, les migrations, puis le healthcheck.

---

## 2. Le SHA relie CI, image et CD

Le lien de confiance entre la CI, l'image Docker et le déploiement est le SHA du commit.

Exemple :

```txt
commit main = abc123
```

La CI construit et pousse :

```txt
ghcr.io/rabiegha/ems-api:abc123
ghcr.io/rabiegha/ems-api:main
```

Le CD lancé depuis `main` utilise aussi :

```txt
github.sha = abc123
```

Il vérifie donc :

```txt
CI verte pour abc123 ?
```

Puis il demande au VPS :

```txt
docker pull ghcr.io/rabiegha/ems-api:abc123
```

Il ne suffit pas qu'une CI soit verte quelque part. Le CD doit vérifier que **la CI verte correspond
au même commit** que l'image à déployer.

---

## 3. Build sur GitHub, pas sur le VPS

Ancien risque :

```txt
le VPS build en prod
-> npm/docker/prisma peut casser sur le serveur
-> déploiement fragile
```

Nouveau modèle :

```txt
GitHub Actions build l'image
GHCR stocke l'image validée
VPS pull l'image déjà construite
```

Le VPS ne fait plus :

```txt
docker build
npm install
npm run build
```

Il fait seulement :

```txt
docker pull ghcr.io/rabiegha/ems-api:<sha>
docker compose up -d --no-build
```

Ce modèle réduit le risque "build cassé en prod".

---

## 4. `workflow_dispatch`

`workflow_dispatch` = bouton manuel dans GitHub Actions.

Chemin :

```txt
Repo GitHub -> Actions -> CD prod -> Run workflow
```

GitHub affiche alors un formulaire :

```txt
confirm = DEPLOY
hotfix = true/false
migration_risky = true/false
```

Ce bouton ne disparaît pas après un déploiement. Il reste disponible tant que le fichier
`.github/workflows/cd-prod.yml` existe sur la branche `main`.

Si `main` n'a pas changé et qu'on relance le workflow, le CD relance le déploiement du même SHA.

---

## 5. `hotfix=true`

`hotfix=true` ne contourne pas la protection de branche, ne force-push rien, et ne saute pas les
tests.

Il sert uniquement à bypasser la garde horaire du CD prod :

```txt
hors 22h-06h Europe/Paris
-> CD prod bloqué

hors 22h-06h + hotfix=true
-> garde horaire bypassée
-> le reste du CD continue normalement
```

Le reste reste actif :

- CI verte obligatoire,
- scan migrations risquées,
- pull image,
- migrations,
- healthcheck,
- rollback si KO.

---

## 6. `migration_risky=true`

`migration_risky` est un bouton d'autorisation, pas une détection.

Par défaut :

```txt
migration_risky=false
```

Sens réel :

```txt
migration_risky=false
  -> je n'autorise pas les migrations destructives

migration_risky=true
  -> j'ai vérifié, j'autorise explicitement cette migration risquée
```

Le CD scanne les migrations SQL nouvelles pour repérer :

```txt
DROP TABLE
DROP COLUMN
ALTER COLUMN
TRUNCATE
```

Règle :

```txt
si migration risquée détectée + migration_risky=false
  -> stop avant de toucher la prod

si migration risquée détectée + migration_risky=true
  -> continue
```

Pourquoi c'est utile :

- `DROP COLUMN` peut supprimer des données encore utiles.
- Prisma peut parfois générer `DROP old_column + ADD new_column` au lieu d'un vrai rename.
- `ALTER COLUMN SET NOT NULL` peut échouer si des lignes existantes contiennent `NULL`.

Le bon réflexe si le CD bloque :

```txt
lire la migration SQL
vérifier backup récent
vérifier impact données
valider que la perte/modification est voulue
relancer avec migration_risky=true seulement si assumé
```

Amélioration souhaitable : ajouter aussi un scan migration risquée en CI sur PR, pour alerter plus
tôt. Le CD doit quand même garder le dernier verrou avant prod.

---

## 7. Les trois accès à ne pas mélanger

```txt
CI -> GHCR push
  Auth : GITHUB_TOKEN automatique GitHub Actions

CD -> VPS SSH
  Auth : VPS_HOST / VPS_USER / VPS_SSH_KEY

VPS -> GHCR pull
  Auth : docker login ghcr.io avec PAT read:packages
```

### `GITHUB_TOKEN`

- généré automatiquement par GitHub Actions à chaque run ;
- temporaire ;
- pas à créer dans les secrets ;
- utilisé par la CI pour push l'image dans GHCR ;
- dans le workflow, il peut apparaître comme `${{ secrets.GITHUB_TOKEN }}`.

### `VPS_SSH_KEY`

- clé SSH durable créée par nous ;
- clé privée stockée dans GitHub Secrets ;
- clé publique ajoutée sur le VPS dans `/home/debian/.ssh/authorized_keys` ;
- utilisée par le CD GitHub Actions pour entrer sur le VPS.

Connexion logique :

```txt
ssh -i VPS_SSH_KEY debian@VPS_HOST
```

`VPS_USER=debian` n'est pas un mot de passe. Le mot de passe `debian` ne doit pas être mis dans
GitHub Secrets.

### PAT GitHub `read:packages`

- token créé dans GitHub Developer Settings ;
- utilisé sur le VPS avec `docker login ghcr.io` ;
- donne au VPS le droit de pull les images GHCR privées ;
- ne sert pas à pousser les images depuis la CI.

Commande côté VPS :

```bash
docker login ghcr.io -u Rabiegha
```

Le PAT est collé comme mot de passe. Docker garde ensuite l'auth localement, généralement dans :

```txt
~/.docker/config.json
```

Il faudra renouveler le PAT à expiration.

---

## 8. GHCR et packages GitHub

GHCR = GitHub Container Registry.

Une image back attendue :

```txt
ghcr.io/rabiegha/ems-api:<sha>
```

Si GitHub -> Rabiegha -> Packages affiche "Get started with GitHub Packages", c'est probablement
qu'aucune image n'a encore été publiée.

Dans le workflow actuel :

```txt
PR vers staging/main
  -> image buildée mais pas poussée

push sur staging/main
  -> image buildée et poussée dans GHCR
```

Donc le package peut rester invisible tant que le workflow n'a jamais réussi sur un vrai push
`staging` ou `main`.

---

## 9. Healthcheck

Le healthcheck est une URL de santé, ici `/health` côté Nest.

Il répond à la question :

```txt
est-ce que l'API est vraiment exploitable ?
```

Le healthcheck enrichi doit exposer :

```json
{
  "status": "ok",
  "db": "ok",
  "redis": "ok",
  "migrations": "ok",
  "version": "abc123"
}
```

Il doit renvoyer HTTP 503 si un check critique est KO.

Dans le CD :

```txt
deploy
-> appel /health
-> si KO, rollback automatique
```

Attention aux chemins :

```txt
code Nest : /health
via nginx/API public : parfois /api/health
```

Il faut vérifier que les scripts, nginx et l'app parlent bien du même endpoint.

---

## 10. Tests event-critical

Les tests actuels sont utiles, mais ne suffisent pas encore à sécuriser les chemins critiques event.

Couverture déjà présente :

- auth basique ;
- refresh token/logout ;
- accès `/users` ;
- accès `/organizations/me` ;
- création utilisateur ;
- smoke front login ;
- QR token en unitaire.

Trous critiques avant event :

- scan QR / validation billet en vrai E2E API ;
- billet déjà scanné ;
- billet inconnu/invalide ;
- token QR modifié ;
- droits scanner/admin/non-autorisé ;
- inscription/registration event ;
- healthcheck réel avec Postgres/Redis dans un E2E.

Priorité raisonnable avant event :

```txt
1. health.e2e-spec.ts
2. scan-qr.e2e-spec.ts
3. permissions-event.e2e-spec.ts
4. registration.e2e-spec.ts si faisable
```

But : ne pas viser 100 % de couverture. Viser les parcours qui casseraient l'exploitation pendant
l'event.

---

## 11. Branch protection

Les protections de branches sont le garde-fou humain autour de la CI/CD.

Pour `main` :

- PR obligatoire ;
- CI obligatoire ;
- 1 approbation obligatoire ;
- force-push interdit ;
- deletion interdite ;
- linear history recommandé.

Pour `staging` :

- PR obligatoire ;
- CI obligatoire ;
- force-push interdit ;
- deletion interdite ;
- approbation optionnelle mais recommandée proche event.

`hotfix=true` ne contourne pas ces protections. Un hotfix propre reste :

```txt
branche hotfix/*
-> PR vers main
-> CI verte
-> merge
-> CD prod manuel avec hotfix=true si hors horaire
```

