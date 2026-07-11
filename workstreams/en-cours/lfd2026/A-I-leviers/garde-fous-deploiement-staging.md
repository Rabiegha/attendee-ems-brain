# Garde-fous déploiement staging (anti-collision prod)

> **Origine :** post-mortem [502 prod du 25/06](../../bugs/fait/2026-06-25-prod-502-collision-compose.md)
> (collision de nom de projet Docker `backend` entre staging et prod sur le même VPS).
> **But :** rendre impossible / détectable immédiatement toute action staging qui toucherait la prod.
> **Créé :** 2026-07-10

---

## Tableau de suivi

| # | Garde-fou | Statut | Note |
|---|---|---|---|
| 1 | **Isolation compose** : `name: ems-staging`, `container_name: ems-staging-*`, réseau dédié `ems-staging-network`, ports bindés `127.0.0.1` | ✅ **Fait** | Fix `5f72daa` + repris/complété dans `infra/staging-stack` (PR #14). Vérifié dans `docker-compose.staging.yml`. |
| 2 | **Répertoire séparé sur le VPS** : checkout staging dans `/opt/ems-attendee/staging/backend/` (clone dédié de la branche `staging`), jamais de fichiers staging dans le dossier prod | ✅ **Fait** (2026-07-10) | Clone créé, `.env.staging*` déplacés hors du dossier prod, stack relancée depuis le nouveau dossier (adoption sans recréation), prod vérifiée 200. Runbook mis à jour. |
| 3 | **Jamais de `docker compose up` « nu »** sur le VPS : toujours `-f docker-compose.staging.yml --env-file .env.staging`, encapsulé dans un script `staging-up.sh` | ⚪ À faire | Écrire le script (up/rebuild api/logs/down) et l'utiliser systématiquement. |
| 4 | **Checklist avant/après chaque déploiement staging** : avant → `docker compose ls` ; après → `docker ps --format '{{.Names}}'` (seuls des `ems-staging-*` ont bougé) + `curl` health **prod** | ⚪ À faire | À intégrer au runbook + idéalement à la fin de `staging-up.sh` (check automatique). |
| 5 | **Limites de ressources** (`cpus`/`mem_limit`) conservées dans le compose staging + runs k6 en **fenêtre creuse** avec `docker stats` prod sous les yeux | ✅ **Fait** (config) | Limites présentes dans le compose — **ne pas les retirer**. La fenêtre creuse reste une discipline à chaque campagne. |
| 6 | **Garde-fou GitHub** : protéger la branche `staging` (*Require a pull request before merging*) — seuls les merges via PR passent, plus de push direct | ⚪ À faire | Aurait bloqué l'incident du 09/07 : un `git pull` sur une vieille `staging` locale a ressuscité l'historique fourre-tout (merge `e2d90ee` poussé directement). À activer **après** la recréation propre de `staging`. |

**Légende :** ✅ fait · 🟡 en cours · ⚪ à faire.

---

## Détail des points à faire

### 2 — Répertoire séparé — ✅ fait le 2026-07-10
- Clone dédié : `/opt/ems-attendee/staging/backend` (branche `staging`, remote `git@github.com:Rabiegha/attendee-ems-back.git`).
- `.env.staging` + 2 backups déplacés du dossier prod vers le clone (`chmod 600`).
- Stack relancée depuis le nouveau dossier : même projet `ems-staging` → conteneurs **adoptés** sans recréation.
- ℹ️ Warning bénin restant : les volumes `ems_staging_*_data` portent le label `project=backend`
  (créés avant le fix `name:`) — retrouvés par **nom explicite**, aucun impact. Disparaîtrait
  seulement en recréant les volumes (= perte des données restaurées, inutile).
- Reste versionné dans le dossier prod (normal, même repo) : `docker-compose.staging.yml`,
  `.env.staging.example` — la règle est de **ne jamais lancer le compose staging depuis là**.

### 3 — Script `staging-up.sh`
- Wrapper qui `cd /opt/ems-attendee/staging/backend` puis lance la commande complète
  (`-f docker-compose.staging.yml --env-file .env.staging`).
- Sous-commandes utiles : `up`, `rebuild-api` (= `up -d --build api`), `logs`, `down`.

### 4 — Checklist avant/après
- **Avant :** `docker compose ls` → repérer les projets existants (`backend` = prod, `ems-staging`).
- **Après :** `docker ps --format '{{.Names}} {{.Status}}'` → vérifier que rien de prod n'a redémarré ;
  `curl -s https://api.attendee.fr/api/health` → 200.
- Intégrable en check automatique en fin de `staging-up.sh` (échoue bruyamment si la prod ne répond pas).

### 6 — Protection de branche GitHub
- GitHub → repo `attendee-ems-back` → *Settings → Branches → Add branch protection rule* → pattern `staging` → cocher **Require a pull request before merging**.
- Contexte : le 09/07, un `git pull` fait sur un clone qui avait encore l'**ancienne** `staging` (fourre-tout archivé) a créé un merge (`e2d90ee`) qui a réintroduit tout l'historique pollué dans la `staging` recréée depuis `main`, puis a été poussé directement. Avec la protection, ce push aurait été **rejeté**.
- ⚠️ Ordre : activer la règle **après** avoir recréé `staging` proprement (la recréation elle-même passe par un force-push qui serait bloqué par la règle).
- Règle complémentaire côté humain : après toute recréation d'une branche partagée, **chaque clone** doit faire `git fetch && git checkout staging && git reset --hard origin/staging` (postes des 2 devs + les 2 checkouts du VPS).
