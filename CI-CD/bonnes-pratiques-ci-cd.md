# Bonnes pratiques CI/CD — guide d'équipe (Rabie + Corentin)

> Rédigé le 2026-07-22, à chaud après l'incident « flip-flop main/staging » du 20/07
> et la CI staging rouge du 21/07. Objectif : que ces deux situations ne se
> reproduisent plus. À lire une fois en entier, puis à garder comme référence.

---

## 1. Le modèle mental : qui fait quoi

```
feature branch ──PR──▶ staging ──(CI verte)──▶ CD staging auto ──▶ staging.attendee.fr
                          │
                          └──PR de merge──▶ main ──(CI verte + action manuelle)──▶ CD prod ──▶ attendee.fr
```

| Branche | Rôle | Qui déploie | Où |
|---|---|---|---|
| `staging` | intégration continue | **CD staging, automatique** à chaque push si CI verte | staging.attendee.fr |
| `main` | miroir exact de la prod | **CD prod, manuel** (`workflow_dispatch` + `confirm=DEPLOY`) | attendee.fr |

**Règle d'or n°1 : `main` doit toujours refléter ce qui tourne en prod.**
Si un jour ce n'est plus vrai (déploiement manuel, hotfix hors flux), c'est un
incident à régulariser immédiatement (voir §6 réconciliation).

**Règle d'or n°2 : le code n'atteint la prod QUE via l'image GHCR construite par la CI.**
Jamais de `docker build` sur le VPS, jamais de `git pull` + rebuild à la main.
L'image `ems-api:local` trouvée en prod le 20/07 est exactement l'anti-pattern.

---

## 2. Le VPS n'est pas un poste de dev

C'est la cause racine du flip-flop du 20/07 : des `git checkout` / manipulations
de branches faites directement dans `/opt/ems-attendee/backend` (prod) et
`/opt/ems-attendee/staging/backend`, jusqu'à ce que le répertoire prod se
retrouve **sur la branche staging** et les conteneurs croisés entre les deux
environnements (d'où aussi le restore-test cassé du 22/07).

**Interdits sur le VPS :**
- `git checkout <branche>`, `git reset`, `git pull` manuel dans les répertoires de déploiement
- `docker build` ou `docker compose up --build`
- lancer un `docker compose` prod depuis le répertoire staging (ou l'inverse)
- modifier `.env` / `.env.staging` sans le noter dans le brain

**Autorisés :** lecture (logs, `docker ps`, `git log`), scripts de maintenance
documentés, et le pattern « manual-fix » ci-dessous **uniquement en incident**.

### Bypass d'urgence (incident uniquement, à documenter dans le brain)
```bash
cd /opt/ems-attendee/staging/backend        # ou backend pour la prod
export ATTENDEE_DEPLOY_SHA=$(cat .deploy/deployed_sha)
export ATTENDEE_STAGING_DEPLOY_GUARD="manual-fix:<raison courte>"   # ou _PROD_
export ATTENDEE_DEPLOY_SOURCE='manual-incident-fix'
export EMS_API_IMAGE="ghcr.io/rabiegha/ems-api:$ATTENDEE_DEPLOY_SHA"
docker compose -p ems-staging -f docker-compose.staging.yml --env-file .env.staging up -d --no-build <services>
```
Les **deploy guards** (variables d'interpolation obligatoires dans les compose,
PRs #59/#60) existent précisément pour qu'un `docker compose up` « naïf » depuis
le mauvais dossier échoue au lieu de casser un environnement en silence.
**Ne jamais les contourner en les exportant « pour que ça passe »** sans raison
d'incident : c'est le garde-fou qui nous a manqué le 20/07.

### Fichiers sacrés
- `.deploy/deployed_sha` : le CD s'en sert pour diff les migrations risquées.
  S'il est faux, le guard de migration compare n'importe quoi.
- Les fichiers non suivis dans les répertoires de déploiement (ex. CSV warmup
  dans `scripts/ops/`) : le CD fait un checkout — vérifier qu'ils survivent ou
  les sauvegarder avant une opération manuelle.

---

## 3. Un changement de contrat API = tests + front dans le même mouvement

Leçon de la CI rouge du 21/07 (`d07c360`) : le scan dupliqué est passé de
`201 (duplicate)` à `409 ALREADY_SESSION_SCANNED`, mais les specs e2e
attendaient toujours 201 → CI staging rouge pendant ~20h → CD staging bloqué
pour **tout le monde**, y compris les commits sains d'après.

**Checklist avant de pousser un changement de contrat (status code, shape de réponse, champ renommé) :**
1. Les specs e2e/unit qui testent ce contrat sont mises à jour **dans le même commit/PR**.
2. `npm run test:e2e` en local (ou au minimum les specs du module touché) avant push.
3. Le front (web + mobile + print-client) consomme-t-il ce contrat ? Si oui :
   ticket ou PR front liée, et stratégie **expand/contract** (le back accepte
   l'ancien ET le nouveau pendant la transition, on ne casse jamais un client
   déjà déployé).
4. Une CI rouge sur `staging` est un **stop-the-line** : la personne qui l'a
   mise au rouge la répare (ou revert) en priorité sur tout le reste. Personne
   ne merge par-dessus une CI rouge.

---

## 4. Ordre de déploiement : back d'abord, front ensuite

Le front est compilé contre le contrat du back. Déployer un front qui appelle
des endpoints pas encore en prod = erreurs utilisateur.

1. **Back** : merge → CD prod → vérifier `/health` + un parcours critique.
2. **Front** : seulement après, et après audit rapide « les nouveaux endpoints
   que ce front appelle existent-ils dans le back déployé ? »
3. Jamais l'inverse, sauf si le changement front est purement cosmétique.

---

## 5. Éviter les conflits git entre nous

Le merge de réconciliation du 22/07 avait **11 fichiers en conflit**, presque
tous évitables. Les pratiques qui les évitent :

- **Petites PRs, mergées vite.** Une branche qui vit 3 jours accumule les
  conflits. Viser < 400 lignes de diff, merger dans les 24h.
- **`staging` → `main` régulièrement** (au moins après chaque chantier terminé,
  idéalement chaque semaine). Plus l'écart grandit (on était à 80 commits !),
  plus la réconciliation est risquée.
- **Se répartir les fichiers chauds.** `public.service.ts`,
  `registrations.service.ts`, `sessions.service.ts` sont les points de
  contention. Si on doit tous les deux y toucher la même semaine : se le dire
  (message ou note dans le brain `workstreams/`), et rebaser souvent.
- **Rebaser sa branche sur `origin/staging` chaque matin** quand un chantier
  est en cours : `git fetch && git rebase origin/staging`. Les conflits se
  règlent petit à petit, pas en bloc à la fin.
- **Un chantier = une branche = une PR.** Pas de commits directs sur `staging`,
  même « juste un petit fix » : ça passe par PR, la CI valide, et l'autre voit
  passer le changement.
- **Fichiers générés/config partagée** (`schema.prisma`, `.env.example`,
  `ci.yml`) : les modifier dans des commits dédiés et les merger vite — ce sont
  les conflits les plus pénibles à résoudre tard.

---

## 6. Réconciliation staging → main (résumé opérationnel)

Méthode complète : [reconciliation/staging-main-2026-07-20.md](reconciliation/staging-main-2026-07-20.md) et [reconciliation/quand-reconcilier.md](reconciliation/quand-reconcilier.md).

1. Worktree jetable : `git worktree add /tmp/reconcile origin/main -b chore/reconcile-...`
2. `git merge origin/staging` → résoudre les conflits **en gardant la version
   la plus récente/transactionnelle (staging en général)**, en vérifiant
   qu'aucune méthode/route propre à `main` n'est perdue
   (`comm -23 <(git show origin/main:fichier | grep ...) <(git show origin/staging:fichier | grep ...)`).
3. Build local + zéro marqueur `<<<<<<<` restant.
4. Push + PR vers `main` → CI complète → merge **explicite** (pas d'auto-merge).
5. CD prod ensuite, jamais avant.

⚠️ **Forward-only** : si des migrations staging sont déjà appliquées en DB prod
(cas du 20/07), on ne revient jamais à un ancien `main` — on avance.

---

## 7. Déployer en prod : la procédure

```bash
# Fenêtre normale : 22h–06h (Paris). En journée : hotfix=true obligatoire.
gh workflow run cd-prod.yml --ref main -f confirm=DEPLOY [-f hotfix=true]
gh run watch <run-id>
```

Après le déploiement, toujours :
- `curl -fsS https://attendee.fr/api/health`
- un login + un parcours critique (liste events, un scan si possible)
- `docker ps` sur le VPS : images `ghcr.io/rabiegha/ems-api:<sha>` (jamais `:local`)
- vérifier `staging.attendee.fr` toujours vivant (les deux partagent le VPS)

**Un déploiement prod se fait à deux de préférence** (un qui déploie, un qui
vérifie), et jamais juste avant de partir.

---

## 8. Post-incident : les 3 réflexes

1. **Documenter dans le brain** (`postmortems/` ou `CI-CD/`) : chronologie,
   cause racine, ce qui a été fait, follow-ups. À chaud, pas « plus tard ».
2. **Vérifier les dégâts collatéraux** : le flip-flop du 20/07 a cassé le
   restore-test (découvert le 22/07) et éjecté un utilisateur (refresh token,
   bug connu `bugs/2026-07-01-eject-mobile-refresh-rotation-499.md`). Un
   incident infra a presque toujours des effets de bord différés.
3. **Transformer en garde-fou** : chaque incident doit laisser derrière lui un
   guard, un test ou une règle dans ce doc (ex. deploy guards après le 20/07).

---

## TL;DR affichable

1. `main` = la prod, toujours. `staging` = intégration, CD auto.
2. Le VPS est en lecture seule pour les humains (hors incident documenté).
3. Jamais de build local en prod ; uniquement les images GHCR de la CI.
4. Changement de contrat ⇒ tests mis à jour dans le même commit + plan front.
5. CI rouge sur staging = stop-the-line, on répare avant tout le reste.
6. Back avant front, toujours.
7. Petites PRs, rebase quotidien, réconciliation staging→main hebdo.
8. Prod : `confirm=DEPLOY`, fenêtre 22h–06h sinon `hotfix=true`, vérifs post-deploy.
