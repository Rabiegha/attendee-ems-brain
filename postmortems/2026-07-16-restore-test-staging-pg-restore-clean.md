# Post-mortem — Restore-test staging rouge du 16/07 (pg_restore --clean sans CASCADE)

> **Statut :** ✅ corrigé (commit `78d7337` sur `main`, `132d025` sur `staging`, 16/07). Post-mortem rédigé le 2026-07-16.
> **Suite (option A) :** ✅ `e56acf9` main / `77b7d52` staging — le restore-test est désormais **isolé dans une base jetable** (`ems_restore_test`), staging live n'est plus jamais impacté (cf. §8).
> **App :** back / ops backup (VPS OVH prod, conteneur `ems-staging-postgres`).
> **Impact :** **aucun sur la prod ni sur les backups** — le dump prod du 16/07 est valide et restaurable. Seul le **test de restauration automatisé** en staging est tombé en échec (alerte email « ECHEC db-restore-staging-test.sh (ligne 69) »).
> **Scripts :** [db-restore-staging-test.sh](../../attendee-ems-back/scripts/backup/db-restore-staging-test.sh) · [db-dump-server.sh](../../attendee-ems-back/scripts/backup/db-dump-server.sh) · [lib-common.sh](../../attendee-ems-back/scripts/backup/lib-common.sh)

---

## 1. Résumé

Le job quotidien `ems-db-restore-test.timer` restaure le dernier dump prod dans la base
**staging** puis lance des smoke tests (« une sauvegarde jamais restaurée n'est pas une
sauvegarde »). Dans la nuit du 16/07, il a échoué à la **ligne 69** (l'appel `pg_restore`).

La cause : le restore utilisait `pg_restore --clean --if-exists`, qui droppe les objets
**un par un sans `CASCADE`**. Dès qu'il existe des dépendances (FK, index/contraintes
partagés, extensions), les `DROP` échouent → les objets restent → les `CREATE` échouent
(« already exists ») → les `COPY` se font **par-dessus** les anciennes données →
`duplicate key value`. Bilan : `pg_restore: warning: errors ignored on restore: 24` →
code retour non-nul → trap `ERR`.

Le backup lui-même était **sain** : les données se chargeaient bien (~132 k inscriptions),
seule la **méthode de restauration du test** était fragile.

## 2. Cause racine

- `pg_restore --clean` sans `CASCADE` ne sait pas dropper un objet qui a des dépendants
  (`cannot drop ... because other objects depend on it`).
- **Facteur déclencheur** : entre le dump du 15/07 (dernier test VERT) et celui du 16/07,
  de **nouvelles FK / objets** ont été ajoutés en prod via les migrations mergées le 15/07
  (`session_checkin_capacity`, `email_events_c2_mailgun`). Ces dépendances supplémentaires
  ont fait basculer les `DROP` de « ça passe » à « ça échoue » → la base staging n'était
  plus jamais vidée proprement → collisions de clés au `COPY`.
- **Angle mort monitoring** : le **stderr de `pg_restore` n'était capturé nulle part**.
  L'alerte email ne disait que « ECHEC ligne 69 », sans la cause. Le diagnostic n'a été
  possible qu'en allant lire le **journal systemd** (`journalctl -u ems-db-restore-test`).

## 3. Timeline (2026-07-16, heures Europe/Paris)

| Heure       | Événement                                                                                                   |
| ----------- | ----------------------------------------------------------------------------------------------------------- |
| 15/07 04:05 | Dernier restore-test **VERT** (RTO 296s).                                                                    |
| 15/07 journée | Merges ajoutant des FK/objets (`session_checkin_capacity`, `email_events_c2_mailgun`).                     |
| 16/07 03:03 | Cycle de backup **OK** : dump prod valide (`pg_restore --list` OK), rotation OK, offsite R2 OK.              |
| 16/07 04:03:31 | Restore-test démarré (checksum du dump vérifié).                                                          |
| 16/07 04:08:27 | **ECHEC ligne 69** : `pg_restore` sort non-nul (`errors ignored on restore: 24`). Alerte email envoyée. |
| 16/07 ~08:39 | 1er run correctif (reset schéma) : restore **VERT** mais smoke test 3 en 503 (`migrations:down`).           |
| 16/07 ~08:52 | 2e run après fix du smoke test 3 : **RESTORE-TEST VERT** complet (RTO 301s).                                 |

## 4. Détection

- **Alerte email automatique** (`on_failure` → `alert_email`) à 04:08 vers `ckistler@choyou.fr`,
  `rgharghar@choyou.fr`. Le mécanisme d'alerte a bien fonctionné.
- **Manque** : l'alerte ne contenait pas la cause (stderr non capturé) → diagnostic manuel
  via `journalctl` nécessaire.

## 5. Correctif appliqué (`78d7337` main / `132d025` staging)

1. **Reset du schéma avant restore** au lieu de `pg_restore --clean` :
   `DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public; GRANT ...` → cible **vierge**
   et déterministe, plus aucune erreur de dépendance ni collision de clés.
2. **`pg_restore --exit-on-error`** (échec net à la 1re vraie erreur) **+ capture du stderr**
   dans un journal dédié `restore-test-<ts>.log` (conservé uniquement en cas d'échec).
3. **Smoke test 3 assoupli** : on valide le composant **`"db":"ok"`** du `/api/health`
   (l'app atteint la base restaurée) au lieu du **status HTTP global**. En restaurant un dump
   **prod** sous du code **staging plus récent**, `"migrations":"down"` (donc status `ko`,
   HTTP 503) est **attendu** et sans rapport avec la restaurabilité.
4. **`db-dump-server.sh`** : même capture de stderr pour `pg_dump` et `pg_restore --list`
   (le dump prod avait le même angle mort).

**Validation en réel** (VPS, 16/07 08:52) :
`Smoke test 1 OK (RTO 301s) · Comptages users=173 events=537 registrations=132229 badges=6280 ·
Smoke test 3 OK (db=ok) · Smoke test 4 OK (132229 lignes) · === RESTORE-TEST VERT ===`.

## 6. Amélioration du monitoring

- Le stderr de `pg_restore`/`pg_dump` est désormais **remonté dans `backup.log`** en cas
  d'échec → **inclus dans l'alerte email** (`on_failure` fait un `tail -n 20` du log).
  Plus besoin de `journalctl` pour connaître la cause d'un échec.
- Journal de restore dédié conservé sur échec pour post-mortem, nettoyé sur succès (pas
  d'accumulation de fichiers).

## 7. Leçons

- **`pg_restore --clean` n'est pas fiable pour re-restaurer par-dessus une base peuplée** dès
  qu'il y a des dépendances : préférer un **reset de schéma** (`DROP SCHEMA ... CASCADE`) ou
  une base fraîche.
- **Toujours capturer le stderr** des commandes critiques sous `set -e` + `trap ERR` : sinon
  l'alerte signale l'échec sans la cause.
- Un **health check applicatif** mélange des signaux (db / redis / migrations) : pour un test
  de restaurabilité, cibler le **composant** pertinent (`db`), pas le status global.
- Le test a **bien joué son rôle** : il a détecté qu'un changement de schéma cassait la
  procédure de restore. La procédure de **DR réelle** (restore dans une base vide) n'était pas
  menacée, mais le test l'a rendu visible tôt.

---

## 8. Suite : alerte Netdata « staging unhealthy » → isolation dans une base jetable (option A)

> **Statut :** ✅ corrigé — commit `e56acf9` (`main`) / `77b7d52` (`staging`, `chore/gotenberg-cloudrun-poc`), 16/07.

### 8.1. Symptôme

Quelques heures après le premier fix, **Netdata** a levé `docker_container_unhealthy` sur
`ems-staging-api` (chart `docker_local.container_ems-staging-api_health_status`), « raised for
18 minutes » puis *Recovered*. Le conteneur oscillait entre `unhealthy` et redémarrages, avec
`/api/health` = `{"status":"ko","db":"ok","redis":"ok","migrations":"down"}` → HTTP 503.

### 8.2. Cause racine (défaut de conception du test)

Le restore-test restaurait le dump prod **dans la base staging LIVE** (`ems_staging`), celle
que sert l'API staging. Résultat : après chaque exécution (04:00), staging tournait sur le
**schéma prod** (en retard sur le code staging) → migration `session_checkin_capacity` non
appliquée → `migrations:down` → status `ko` → **HTTP 503** → healthcheck Docker `unhealthy` →
alerte Netdata. Le signal `db:ok`/`redis:ok` confirmait que seule la couche migrations était
en cause. **Ce n'était pas un incident applicatif** mais un effet de bord du test non nettoyé.

### 8.3. Remise en état immédiate (+ piège pgbouncer)

Pour re-verdir staging il fallait appliquer la migration manquante. `prisma migrate deploy` a
d'abord échoué avec `P1002 — Timed out trying to acquire a postgres advisory lock
(pg_advisory_lock(72707369))` : Prisma passait par **pgbouncer** (pooling *transaction*), qui
conserve le lock de **session** sur une connexion backend jamais relâchée → toutes les
tentatives suivantes se bloquaient (même en connexion directe). Résolution :

1. Identifier le détenteur (`granted=t`, `state=idle`, dernière requête `COMMIT`) via `pg_locks`.
2. Le libérer : `SELECT pg_terminate_backend(pid) FROM pg_locks WHERE locktype='advisory' AND objid=72707369;`.
3. Rejouer la migration en **connexion directe** (variable `DIRECT_URL` → `postgres:5432`, hors pgbouncer) :
   `docker exec ems-staging-api sh -c 'cd /app && DATABASE_URL="$DIRECT_URL" npx prisma migrate deploy'`.

Résultat : `/api/health` = `status:ok, migrations:ok`, HTTP 200, conteneur `healthy` (fails=0).

### 8.4. Correctif de fond — option A (base jetable)

Le restore-test restaure désormais dans une **base jetable dédiée** `ems_restore_test`
(créée par `CREATE DATABASE` puis supprimée en fin de run), **jamais** dans `ems_staging` :

- Garde-fous : refus si `RESTORE_DB` = base staging live, ou si le nom ne ressemble pas à une
  base jetable (`*restore*`/`*test*`).
- **Smoke test 3 revu** : comme la base restaurée n'est plus branchée à l'API staging, le test
  exécute une **requête applicative représentative** directement sur `ems_restore_test`
  (dernier event + volume d'inscriptions) au lieu d'interroger `/api/health`.
- **Preuve de non-intrusivité** ajoutée : on vérifie que l'API staging live répond **toujours
  200** après le test (non bloquant).
- **Nettoyage** : la base jetable est supprimée après succès (espace libéré, recréée au run suivant).

**Validation en réel** (VPS, 16/07 09:21) :
`Smoke test 1 OK (RTO 301s) · users=173 events=537 registrations=132229 badges=6280 ·
Smoke test 3 OK (dernier event → inscriptions=615) · Smoke test 4 OK (132229 lignes) ·
Non-intrusivité OK : API staging live toujours saine (HTTP 200) · base jetable supprimée ·
=== RESTORE-TEST VERT ===`. Staging live est **resté 200 pendant tout le test**.

### 8.5. Leçons complémentaires

- **Ne jamais restaurer un test dans une base LIVE** : utiliser une base jetable, sinon
  l'appli branchée dessus part en 503 (migrations désalignées) jusqu'à re-migration.
- **Migrations Prisma : toujours via connexion directe** (`DIRECT_URL`), jamais via pgbouncer
  (les advisory locks de session survivent au pooling et bloquent les migrations suivantes).
- Une alerte `container_unhealthy` juste après la fenêtre de backup (04:00) doit faire
  suspecter un effet de bord du restore-test avant de conclure à un incident applicatif.
