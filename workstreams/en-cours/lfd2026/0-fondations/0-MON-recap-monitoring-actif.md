# 0-MON — Récap monitoring actif

> Date : 2026-07-16 · Statut : ✅ monitoring prod opérationnel
> Références : [0-MON-plan.md](./0-MON-plan.md) · [finalisation-0-mon.md](./finalisation-0-mon.md) ·
> [runbook Netdata](../../../ops/netdata-monitoring.md)

## 1. Ce qu'on surveille

Le monitoring couvre maintenant 4 niveaux :

| Niveau | Outil | Rôle |
| --- | --- | --- |
| Uptime API | sonde externe | Vérifie que l'API répond sur `/api/health` |
| Erreurs applicatives | Sentry | Remonte les exceptions back/front avec contexte |
| Système VPS | Netdata | CPU, RAM, disque, I/O, conteneurs Docker |
| Files async | `/health/queues` + timer systemd | Surveille les files BullMQ qui gonflent |

Objectif : ne plus découvrir les incidents par hasard. En priorité, être alerté avant le scénario
critique : **disque plein → Postgres ne peut plus écrire le WAL → API down**.

## 2. Accès dashboard

Dashboard Netdata :

```text
https://api.attendee.fr/netdata/
```

L'accès est protégé par basic auth nginx. Les identifiants sont stockés localement dans :

```text
local-files/accounts/netdata-production.md
```

Le port direct Netdata est volontairement fermé :

```text
http://51.75.252.74:19999 → ne doit PAS répondre
```

Netdata écoute seulement sur le VPS :

```text
127.0.0.1:19999   → tunnel SSH / checks locaux
172.17.0.1:19999  → nginx Docker vers /netdata/
```

## 3. Alertes actives

Alertes Netdata EMS chargées :

| Alarme | Cible | Rôle |
| --- | --- | --- |
| `ems_disk_usage` | disque | warning > 70 %, critique > 85 % |
| `ems_disk_usage_80` | disque | palier distinct > 80 % |
| `ems_cpu_sustained` | CPU | CPU moyen élevé sur 10 min |
| `ems_ram_usage` | RAM | RAM utilisée hors cache |

Canal d'alerte :

```text
Discord — webhook EMS Monitoring
```

Le secret webhook est sur le VPS dans :

```text
/opt/ems-attendee/backend/.env.monitoring
```

Il ne doit pas être committé.

## 4. Test réel effectué

Le 2026-07-16, une alerte disque temporaire a été créée puis supprimée :

```text
ems_disk_webhook_test
```

Elle ciblait `disk_space./`, est passée en `WARNING`, a envoyé l'alerte Discord, puis a été nettoyée.

Checks validés :

```text
https://api.attendee.fr/api/health       → 200
https://staging.attendee.fr/api/health   → 200
https://api.attendee.fr/netdata/         → 401 sans auth
http://51.75.252.74:19999                → fermé / 000
```

## 5. BullMQ

Les files BullMQ sont exposées par :

```text
https://api.attendee.fr/api/health/queues
```

Exemple d'état attendu :

```json
{
  "status": "ok",
  "totalWaiting": 0
}
```

Surveillance installée via systemd, car `crontab` n'est pas disponible sur le VPS :

```text
ems-bullmq-check.timer
ems-bullmq-check.service
```

Fréquence :

```text
toutes les 5 minutes
```

Script exécuté :

```text
/opt/ems-attendee/backend/monitoring/check-bullmq-queues.sh
```

Log :

```text
/var/log/ems-bullmq-check.log
```

## 6. Scripts versionnés

Les corrections faites sur le VPS sont maintenant versionnées dans `attendee-ems-back` :

```text
monitoring/apply-netdata-system-bind.sh
monitoring/apply-netdata-system-health.sh
monitoring/check-bullmq-queues.sh
monitoring/systemd/ems-bullmq-check.service
monitoring/systemd/ems-bullmq-check.timer
```

Commandes de vérification sur le VPS :

```bash
cd /opt/ems-attendee/backend
./monitoring/apply-netdata-system-bind.sh --check
./monitoring/apply-netdata-system-health.sh --check
systemctl list-timers --all ems-bullmq-check.timer --no-pager
```

## 7. Logs Docker

Rotation Docker vérifiée sur :

```text
ems-api
ems-nginx
ems-postgres
ems-redis
```

Paramètres :

```text
json-file
max-size = 50m
max-file = 3
```

## 8. Point d'attention nginx

Le conteneur `ems-nginx` sert aussi la conf staging. Après recréation, il doit donc être attaché à :

```text
ems-network
ems-staging-network
```

Sinon `staging.attendee.fr.conf` ne peut plus résoudre :

```text
ems-staging-api
ems-staging-jeu
```

Ce point est versionné dans `docker-compose.prod.yml`.

## 9. Statut final

0-MON est considéré comme **terminé** :

- Sentry activé et testé ;
- Netdata actif, sécurisé, alertes chargées ;
- webhook Discord branché ;
- alerte disque testée en réel ;
- BullMQ surveillé toutes les 5 minutes ;
- rotation Docker vérifiée ;
- suivi chantier passé à 100 %.
