# Finalisation 0-MON (monitoring) et livraison

> Date : 2026-07-14 · Chantier : **0-MON** ([plan](./0-MON-plan.md) · [suivi ligne 0-MON](../03-suivi-chantiers.md))
> Même logique que [finalisation-ci-cd-et-livraison.md](./finalisation-ci-cd-et-livraison.md) :
> checklist d'exécution ordonnée, cochée au fur et à mesure, avec les commandes exactes.
>
> **État au 14/07 : LE CODE EST 100 % FAIT** (branche `chore/monitoring` back, +2 commits E2E du 14/07).
> **Tout ce qui reste est de l'ops VPS** — bloqué par 3 prérequis côté Rabie (§1).

---

## 0. Ce qui est déjà fait (ne pas refaire)

| Brique                                        | État                         | Où                                                                                                     |
| --------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| Netdata compose (name `ems-monitoring`)       | ✅ code                      | `attendee-ems-back/docker-compose.monitoring.yml` (dashboard 127.0.0.1:19999, non exposé — tunnel SSH) |
| Alertes disque 70/80/85 % + CPU/RAM           | ✅ code                      | `monitoring/netdata/health.d/ems-disk.conf`, `ems-cpu-ram.conf`                                        |
| Notifications webhook (Slack-compatible)      | ✅ code                      | `monitoring/netdata/health_alarm_notify.conf` (custom_sender, lit `NETDATA_WEBHOOK_URL`)               |
| Métrique files BullMQ                         | ✅ code + **E2E 14/07**      | endpoint `/health/queues` (`src/health/health.controller.ts`) + `test/health.e2e-spec.ts`              |
| Cron de surveillance BullMQ                   | ✅ code                      | `monitoring/check-bullmq-queues.sh` (alerte si waiting gonfle)                                         |
| Rotation logs Docker (json-file 50m × 3)      | ✅ code                      | `docker-compose.prod.yml` (mergé)                                                                      |
| Sentry back (init si `SENTRY_DSN`)            | ✅ code                      | `src/instrument.ts` — s'active tout seul dès que le DSN est posé                                       |
| Sentry front                                  | ✅ code                      | `src/shared/lib/logger.ts` + `@sentry/vite-plugin` (source maps prod)                                  |
| Uptime externe sur `/health` + alerte         | ✅ **déployé**               | fait par Rabie (avant le 11/07)                                                                        |
| Healthcheck enrichi `/health` (db/redis/migr) | ✅ **en prod**               | via 0-CI — consommé par le CD et l'uptime                                                              |
| Disque VPS                                    | ✅ nettoyé 76 → 44 % (13/07) | reste le legacy 912 Mo à purger (checklist 0-CI §4)                                                    |

## 1. Prérequis côté Rabie (bloquants — rien ne se déploie sans)

- [ ] **Clé SSH VPS sur le MacBook** 🔴 LE bloquant n°1 (vérifié 14/07 : `BatchMode` → `Permission denied`,
      la clé n'existe que sur l'iMac). 2 minutes :

  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/ems_vps -C "rabie-macbook-ems" -N ""
  # puis (mot de passe demandé une dernière fois) :
  ssh-copy-id -i ~/.ssh/ems_vps.pub debian@51.75.252.74
  # et dans ~/.ssh/config :
  #   Host ems-vps
  #     HostName 51.75.252.74
  #     User debian
  #     IdentityFile ~/.ssh/ems_vps
  ```

- [x] **Canal d'alerte choisi + webhook créé** (Slack incoming webhook ou équivalent email→webhook).
      Réutiliser le canal de l'uptime si possible. → la valeur ira dans `.env.monitoring` sur le VPS,
      **jamais dans le chat**.
- [ ] **Projets Sentry créés** (1 back + 1 front, ou 1 projet 2 environnements) → récupérer les **DSN**.
      À poser directement sur le VPS (`.env.production` / `.env.staging`) et en secret GitHub si besoin
      (`gh secret set SENTRY_DSN`), **jamais dans le chat**.

## 2. Déploiement Netdata + alertes disque/CPU (dès le prérequis SSH levé)

- [x] Pousser `chore/monitoring` → PR → `staging` (la CI valide les 31 E2E) — le dossier `monitoring/`
      et le compose arrivent sur le VPS par le `git pull` du checkout.
- [x] Sur le VPS : Netdata existait déjà en **service système** (`netdata v2.10.3`, claim Netdata Cloud).
      Le port direct public `:19999` a été fermé le 16/07 : Netdata écoute uniquement sur
      `127.0.0.1:19999` et `172.17.0.1:19999`.

  ```bash
  cd /opt/ems-attendee/backend
  ./monitoring/apply-netdata-system-bind.sh --check
  ./monitoring/apply-netdata-system-health.sh --check
  ```

- [x] Vérifier le dashboard : `https://api.attendee.fr/netdata/` répond `401` sans auth, `200` avec auth.
- [x] Vérifier que les seuils custom sont chargés : `ems_disk_usage`, `ems_disk_usage_80`,
      `ems_cpu_sustained`, `ems_ram_usage` présents dans l'API Netdata.
- [ ] ⚠️ Leçon 0-CI : `name: ems-monitoring` isole le projet compose — ne PAS lancer depuis un autre
      dossier ni sans le `-f` explicite (incident 502 du 25/06).
      Note 16/07 : prod utilise pour l'instant le service système Netdata ; les scripts
      `apply-netdata-system-*` versionnent cet état.

## 3. Test d'alerte EN RÉEL (ne pas sauter — une alerte jamais testée n'existe pas)

- [x] Créer le canal d'alerte et poser sur le VPS :

  ```bash
  cd /opt/ems-attendee/backend
  echo 'NETDATA_WEBHOOK_URL=<le webhook>' > .env.monitoring
  chmod 600 .env.monitoring
  ./monitoring/apply-netdata-system-health.sh
  ```

- [x] Tester une alerte disque réelle : alarme temporaire `ems_disk_webhook_test` sur `disk_space./`,
      passée en `WARNING` à 71.4 %, puis supprimée et Netdata redémarré.
- [x] Consigner ici la date du test et le canal : **2026-07-16 — Discord `EMS Monitoring`**.

## 4. Sentry actif (config pure, zéro code)

- [x] Poser `SENTRY_DSN` (+ `SENTRY_ENVIRONMENT=production`) dans `.env.production` VPS → recreate API.
- [x] Idem staging (`SENTRY_ENVIRONMENT=staging`).
- [x] Front : `SENTRY_DSN` en secret GitHub → le build CI injecte + upload source maps.
- [x] **Vérifier qu'un event remonte** (back : déclencher une exception de test ; front : idem).
      Tant que ce n'est pas vu dans Sentry, ce n'est pas fait.

## 5. Cron BullMQ + garde-fous disque

- [x] Installer la surveillance BullMQ sur le VPS.
      Note 16/07 : `crontab` absent sur le VPS ; remplacé par un timer systemd versionné.

  ```bash
  systemctl list-timers --all ems-bullmq-check.timer --no-pager
  ```

- [x] Vérifier `/health/queues` en prod : `curl -s https://api.attendee.fr/api/health/queues | jq`
- [x] Rotation logs : vérifier qu'elle est effective (`docker inspect ems-api | grep -A3 LogConfig`).
      Vérifié 16/07 : `ems-api`, `ems-nginx`, `ems-postgres`, `ems-redis` en `json-file max-size=50m max-file=3`.
- [ ] Purge legacy 912 Mo (checklist [0-CI §4](./finalisation-ci-cd-et-livraison.md)) — libère la marge disque.

## 6. Clôture du chantier

- [x] Mettre la ligne **0-MON à 100 %** dans le [suivi](../03-suivi-chantiers.md) UNIQUEMENT quand :
      Netdata up + alerte disque **testée en réel** + Sentry reçoit des events (back ET front) +
      cron BullMQ en place.
- [x] Reporter la config dans la checklist K (résilience event) : qui reçoit les alertes pendant l'event,
      qui a accès au dashboard.

---

## Résumé de l'ordre

```txt
1 (prérequis Rabie : clé SSH ▸ webhook ▸ DSN) → 2 (Netdata) → 3 (test alerte réel)
                                              → 4 (Sentry) en parallèle de 2-3
                                              → 5 (cron BullMQ + garde-fous) → 6 (clôture)
```

**Résultat cible** : être **prévenu avant la panne** — alerte disque avant que Postgres bloque le WAL
(protection n°1), alerte CPU soutenu, alerte file email qui gonfle, erreurs back+front visibles dans
Sentry, dashboard temps réel via tunnel SSH. Zéro nouveau service exposé publiquement.
