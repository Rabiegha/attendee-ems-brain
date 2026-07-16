# Finalisation 0-MON (monitoring) et livraison

> Date : 2026-07-14 · mise à jour : 2026-07-16 soir · Chantier : **0-MON**
> ([plan](./0-MON-plan.md) · [suivi ligne 0-MON](../03-suivi-chantiers.md))
> Même logique que [finalisation-ci-cd-et-livraison.md](./finalisation-ci-cd-et-livraison.md) :
> checklist d'exécution ordonnée, cochée au fur et à mesure, avec les commandes exactes.
>
> **État au 16/07 soir : LE CODE EST 100 % FAIT ET MERGÉ `staging`** (`attendee-ems-back`
> `63c2bd6`, merge de `chore/monitoring`).
> **Tout ce qui reste est de l'ops VPS** — pull/déploiement, activation Netdata/timer BullMQ,
> test d'alerte réel et confirmation Sentry.

---

## 0. Ce qui est déjà fait (ne pas refaire)

| Brique                                        | État                         | Où                                                                                                     |
| --------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| Netdata compose (name `ems-monitoring`)       | ✅ code                      | `attendee-ems-back/docker-compose.monitoring.yml` (dashboard 127.0.0.1:19999, non exposé — tunnel SSH) |
| Alertes disque 70/80/85 % + CPU/RAM           | ✅ code                      | `monitoring/netdata/health.d/ems-disk.conf`, `ems-cpu-ram.conf`                                        |
| Notifications webhook (Slack/Discord-compatible) | ✅ code mergé `staging`   | `monitoring/netdata/health_alarm_notify.conf` (custom_sender, lit `NETDATA_WEBHOOK_URL`/`WEBHOOK_URL`) |
| Métrique files BullMQ                         | ✅ code + **E2E 14/07**      | endpoint `/health/queues` (`src/health/health.controller.ts`) + `test/health.e2e-spec.ts`              |
| Timer systemd de surveillance BullMQ          | ✅ code mergé `staging`      | `monitoring/systemd/ems-bullmq-check.*` + `monitoring/check-bullmq-queues.sh` (alerte si waiting gonfle) |
| Rotation logs Docker (json-file 50m × 3)      | ✅ code                      | `docker-compose.prod.yml` (mergé)                                                                      |
| Sentry back (init si `SENTRY_DSN`)            | ✅ code                      | `src/instrument.ts` — s'active tout seul dès que le DSN est posé                                       |
| Sentry front                                  | ✅ code                      | `src/shared/lib/logger.ts` + `@sentry/vite-plugin` (source maps prod)                                  |
| Uptime externe sur `/health` + alerte         | ✅ **déployé**               | fait par Rabie (avant le 11/07)                                                                        |
| Healthcheck enrichi `/health` (db/redis/migr) | ✅ **en prod**               | via 0-CI — consommé par le CD et l'uptime                                                              |
| Disque VPS                                    | ✅ nettoyé 76 → 44 % (13/07) | reste le legacy 912 Mo à purger (checklist 0-CI §4)                                                    |

## 1. Prérequis côté Rabie (bloquants — rien ne se déploie sans)

- [~] **Clé SSH VPS sur le MacBook** — vérif 16/07 : la clé `~/.ssh/ems_staging_ed25519`
      fonctionne vers `debian@51.75.252.74`, mais l'alias `ems-vps` n'existe pas encore
      dans `~/.ssh/config`. À faire : ajouter l'alias pour éviter les commandes fragiles.

  ```bash
  # dans ~/.ssh/config :
  #   Host ems-vps
  #     HostName 51.75.252.74
  #     User debian
  #     IdentityFile ~/.ssh/ems_staging_ed25519
  ```

- [x] **Canal d'alerte choisi + webhook créé** — vérif 16/07 : `.env.monitoring` présent sur le VPS
      (valeur non affichée). Rabie a reçu au moins une alerte disque.
- [x] **Projets Sentry créés + DSN posés** — vérif 16/07 : `SENTRY_DSN` présent dans les env prod/staging
      et injecté dans les conteneurs `ems-api` / `ems-staging-api`. Reste à confirmer visuellement qu'un
      event remonte bien dans Sentry back + front.

## 2. Déploiement Netdata + alertes disque/CPU (dès le prérequis SSH levé)

- [x] Pousser `chore/monitoring` → `staging` — fait le 16/07 soir (`attendee-ems-back`
      `63c2bd6`). Le dossier `monitoring/` et le compose arriveront sur le VPS par le `git pull`
      du checkout.
- [ ] Sur le VPS :

  ```bash
  cd /opt/ems-attendee/backend
  git pull --ff-only
  ./monitoring/apply-netdata-system-bind.sh
  ./monitoring/apply-netdata-system-health.sh
  ```

- [ ] Vérifier le dashboard : `ssh -L 19999:127.0.0.1:19999 ems-vps` → http://localhost:19999
- [ ] Vérifier que les seuils custom sont chargés : onglet Alerts → `ems_disk_*` présents.
- [ ] ⚠️ Leçon 0-CI : `name: ems-monitoring` isole le projet compose — ne PAS lancer depuis un autre
      dossier ni sans le `-f` explicite (incident 502 du 25/06).

> Vérif 16/07 : `docker ps -a` ne montre aucun conteneur Netdata/monitoring et
> `docker compose -f docker-compose.monitoring.yml --env-file .env.monitoring ps` est vide.
> Conclusion : Netdata n'est pas lancé actuellement, même si `.env.monitoring` et le compose sont présents.

## 3. Test d'alerte EN RÉEL (ne pas sauter — une alerte jamais testée n'existe pas)

- [~] Baisser temporairement le seuil warning disque à ≤ 40 % (le disque est à ~44 %) dans
      `monitoring/netdata/health.d/ems-disk.conf` sur le VPS → `docker restart ems-netdata` →
      **l'alerte doit arriver sur le canal choisi** → remettre le seuil → restart.
- [ ] Consigner ici la date du test et le canal : **alerte disque reçue par Rabie, à rattacher au test exact
      et au canal utilisé**.

## 4. Sentry actif (config pure, zéro code)

- [x] Poser `SENTRY_DSN` (+ `SENTRY_ENVIRONMENT=production`) dans `.env.production` VPS → vérif 16/07 :
      présent dans `ems-api`.
- [x] Idem staging (`SENTRY_ENVIRONMENT=staging`) → vérif 16/07 : présent dans `ems-staging-api`.
- [ ] Front : `SENTRY_DSN` en secret GitHub → le build CI injecte + upload source maps.
- [ ] **Vérifier qu'un event remonte** (back : déclencher une exception de test ; front : idem).
      Tant que ce n'est pas vu dans Sentry, ce n'est pas fait.

## 5. Cron BullMQ + garde-fous disque

- [ ] Installer/activer le timer systemd BullMQ sur le VPS :

  ```bash
  sudo install -m 0644 monitoring/systemd/ems-bullmq-check.service /etc/systemd/system/ems-bullmq-check.service
  sudo install -m 0644 monitoring/systemd/ems-bullmq-check.timer /etc/systemd/system/ems-bullmq-check.timer
  sudo systemctl daemon-reload
  sudo systemctl enable --now ems-bullmq-check.timer
  sudo systemctl list-timers ems-bullmq-check.timer
  ```

- [ ] Vérifier `/health/queues` en prod : `curl -s https://api.attendee.fr/api/health/queues | jq`
- [ ] Rotation logs : vérifier qu'elle est effective (`docker inspect ems-api | grep -A3 LogConfig`).
- [ ] Purge legacy 912 Mo (checklist [0-CI §4](./finalisation-ci-cd-et-livraison.md)) — libère la marge disque.

> Vérif 16/07 :
> - `/api/health/queues` prod OK, toutes les queues à 0 (`email.send`, exports).
> - Aucun ancien cron BullMQ trouvé via `sudo crontab -l` ni `/etc/cron*`.
> - `/opt/ems-attendee/backend/frontend` existe toujours et pèse 912M.

## 6. Clôture du chantier

- [ ] Mettre la ligne **0-MON à 100 %** dans le [suivi](../03-suivi-chantiers.md) UNIQUEMENT quand :
      Netdata up + alerte disque **testée en réel** + Sentry reçoit des events (back ET front) +
      timer BullMQ en place.
- [ ] Reporter la config dans la checklist K (résilience event) : qui reçoit les alertes pendant l'event,
      qui a accès au dashboard.

---

## Résumé de l'ordre

```txt
1 (prérequis Rabie : clé SSH ▸ webhook ▸ DSN) → 2 (Netdata) → 3 (test alerte réel)
                                              → 4 (Sentry) en parallèle de 2-3
                                              → 5 (timer BullMQ + garde-fous) → 6 (clôture)
```

**Résultat cible** : être **prévenu avant la panne** — alerte disque avant que Postgres bloque le WAL
(protection n°1), alerte CPU soutenu, alerte file email qui gonfle, erreurs back+front visibles dans
Sentry, dashboard temps réel via tunnel SSH. Zéro nouveau service exposé publiquement.
