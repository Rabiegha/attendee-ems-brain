# Réconciliation `staging` → `main` — état au 2026-07-20 (`attendee-ems-back`)

> Doc d'action. Décrit **la réconciliation concrète à faire maintenant** entre
> `staging` et `main` sur `attendee-ems-back` : pourquoi, ce qui diverge, les
> conflits exacts, et la méthode pour la mener à bien. Pour le concept général
> (quand ouvrir une branche dédiée), voir [quand-reconcilier.md](./quand-reconcilier.md).

## Pourquoi cette réconciliation est nécessaire maintenant

- Un hotfix email (`isAdminSource` inversé + `EMAIL_TRANSPORT=mailgun`) a été
  déployé en urgence sur `staging` **et** directement sur le serveur de prod
  (checkout d'un commit de `staging` en HEAD détachée, hors flux `main`).
- Le code source de prod est donc à jour sur l'email, mais **le repo Git de
  prod n'est pas proprement synchronisé avec `main`** : il tourne sur un commit
  qui n'existe que sur `origin/staging`.
- Il faut donc, à un moment donné, remettre de l'ordre : soit merger `staging`
  dans `main` (et re-baser proprement le déploiement prod dessus), soit a minima
  documenter/valider cet état avant de continuer à développer sur les deux branches.

## Point de divergence

```
Merge-base (dernier ancêtre commun) : 681c930e2738dc2f113e9be8e8904e7c06a80705
```

| | Commits en avance | Contenu principal |
|---|---|---|
| `staging` en avance sur `main` | 32 | L9a (capacité atomique event, PR #39, très récent), L9b (transaction session slim, PR #33/#36/#37/#38), tooling warm-up/load-test, **fix email `a4d193c`** + approval email session (`86806d9`), prisma directUrl (un chemin différent de celui de `main`) |
| `main` en avance sur `staging` | 12 | Fixes CI/CD (healthcheck, nginx reload), fixes backup/restore, prisma directUrl **via PR #30/#31** (chemin différent, jamais réconcilié avec celui de staging) |

## Conflits réels (simulation `git merge --no-commit --no-ff`, aucun push)

5 fichiers en conflit lors d'un merge `origin/staging` → `origin/main` :

| Fichier | Nature | Gravité |
|---|---|---|
| `prisma/schema.prisma` | Ligne vide en fin de fichier (les deux branches ont ajouté `directUrl` indépendamment) | Trivial |
| `src/modules/sessions/sessions.service.ts` | Refactor du compteur de session (L9b) vs ancien code | Réel — logique métier |
| `src/modules/public/public.service.ts` | Injection de `SessionRegistrationCounterService` (nouveau service, absent de `main`) — 6 blocs de conflit | Réel — logique métier |
| `src/modules/registrations/registrations.service.ts` | 1 conflit (~L1626) : mise à jour des choix de session — ancienne version (stats websocket inline) vs nouvelle version transactionnelle | Réel — logique métier |
| `test/sessions-h.e2e-spec.ts` | Tests du même refactor | Lié au refactor sessions |

✅ Le fix email (`isAdminSource`, `a4d193c`) et l'ajout `directUrl` dans le schéma
Prisma **se fusionnent sans conflit** — ils ne sont donc pas un facteur bloquant
pour cette réconciliation.

⚠️ Tous les conflits réels viennent du **même chantier** : L9a/L9b (transactions
d'inscription event/session), qualifié de *"vraie chirurgie de code"* — nécessite
des tests de non-régression (pas de doublons, quotas respectés, cohérence des
compteurs), pas une résolution mécanique.

## Précédent existant à reprendre

Une tentative a déjà eu lieu le 17/07/2026 : PR **#34** (merge direct) → fermée
faute de résolution propre → remplacée par **`chore/reconcile-staging-main`**
(PR **#35**), conflits résolus une fois en local, CI verte, **jamais mergée**
(décision explicite requise, `main` déclenche la chaîne CD prod).

→ **Reprendre/rafraîchir cette branche** plutôt que d'en ouvrir une nouvelle,
sauf si son historique est trop obsolète par rapport aux 32 commits actuels de
`staging` (à vérifier au moment de la reprise).

## Comment faire la réconciliation

### 1. Rafraîchir l'état des lieux

```bash
cd attendee-ems-back
git fetch origin --prune
git log origin/main..origin/staging --oneline   # doit encore lister ~32 commits
git log origin/staging..origin/main --oneline   # doit encore lister ~12 commits
```

### 2. Vérifier si `chore/reconcile-staging-main` existe encore et son état

```bash
git branch -r | grep reconcile
git log origin/main..origin/chore/reconcile-staging-main --oneline
```

- Si la branche existe et n'a pas trop de retard sur `staging` → la rebaser dessus.
- Sinon → repartir d'une branche neuve depuis `origin/main`.

### 3. Simuler le merge avant tout travail réel (aucun risque, aucun push)

```bash
git worktree add -f /tmp/merge-test origin/main --detach
cd /tmp/merge-test
git merge origin/staging --no-commit --no-ff
grep -rln "^<<<<<<<" .   # liste les fichiers en conflit à ce moment-là
```

Comparer avec la liste des 5 fichiers ci-dessus — si de nouveaux fichiers
apparaissent, un nouveau chantier a divergé entre-temps, à documenter ici.

### 4. Résoudre les conflits dans une vraie branche de travail (pas le worktree jetable)

```bash
git checkout -b chore/reconcile-staging-main origin/main   # ou rebase l'existante
git merge origin/staging --no-ff
# résoudre fichier par fichier :
#  - prisma/schema.prisma          : accepter les deux ajouts directUrl, nettoyer les doublons
#  - sessions.service.ts           : garder la version transactionnelle (staging),
#                                    vérifier qu'aucun appelant de main ne dépend de l'ancienne API
#  - public.service.ts             : injecter SessionRegistrationCounterService partout où requis
#  - registrations.service.ts      : reprendre la version transactionnelle (staging) qui inclut
#                                    déjà l'émission des stats websocket via le compteur
#  - sessions-h.e2e-spec.ts        : garder les tests staging, ajouter ceux de main si absents
```

### 5. Valider avant de proposer le merge vers `main`

- [ ] Build local (`npm run build`) sans erreur TypeScript.
- [ ] Tests unitaires + e2e (`npm test`, `npm run test:e2e`) verts, en particulier
      `sessions-h.e2e-spec.ts` (compteurs, concurrence).
- [ ] Test manuel de non-régression sur l'inscription event ET session (capacité,
      pas de doublon, quota respecté) — cf. workstream L9a/L9b dans
      `workstreams/en-cours/lfd2026/A-I-leviers/01-suivi-leviers.md`.
- [ ] CI GitHub Actions verte sur la branche de réconciliation.
- [ ] Déployer et valider sur **staging** avant tout merge vers `main` (la branche
      de réconciliation contient déjà tout `staging`, donc un déploiement staging
      doit rester strictement équivalent au staging actuel).

### 6. Merge vers `main` — décision explicite requise

Ne pas merger sans validation explicite d'un humain : `main` peut déclencher la
chaîne CD prod. Une fois mergé, aligner le repo Git du serveur de prod (actuellement
en HEAD détachée sur un commit `staging`) sur `main` pour repartir sur un flux propre.
