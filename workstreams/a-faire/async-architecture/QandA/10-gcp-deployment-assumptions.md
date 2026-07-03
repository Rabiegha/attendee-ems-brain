# 10 — Hypothèses de déploiement GCP

> Ce fichier ne propose **pas** de plan GCP final.
> Il liste ce qui est connu depuis le code, ce qui est inconnu, et ce que l'équipe doit décider.

## 1. Connu depuis le code

| Élément | Source | Statut |
|---|---|---|
| Containerisé | [Dockerfile](../../../../../attendee-ems-back/Dockerfile), [Dockerfile.dev](../../../../../attendee-ems-back/Dockerfile.dev), [docker-compose.prod.yml](../../../../../attendee-ems-back/docker-compose.prod.yml), [docker-compose.dev.yml](../../../../../attendee-ems-back/docker-compose.dev.yml) | `FOUND_IN_CODE` |
| Dossier dédié migration | [docs/migration-GCP/](../../../../backlog/a-faire/other/migrations and refactoring/migration-GCP) | `FOUND_IN_CODE` (à relire si pertinent) |
| Variables Redis (`REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_DB`, `QUEUE_KEY_PREFIX`) | `src/config/config.service.ts` | `FOUND_IN_CODE` |
| Variables R2 (`R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_PUBLIC_URL`) | idem | `FOUND_IN_CODE` |
| SMTP (`SMTP_HOST/PORT/USER/PASSWORD/SECURE/FROM`) | idem | `FOUND_IN_CODE` |
| Sentry (`SENTRY_DSN`, `SENTRY_ENVIRONMENT`) | `instrument.ts`, config | `FOUND_IN_CODE` |
| Postgres via Prisma | `prisma/schema.prisma` | `FOUND_IN_CODE` |
| `puppeteer` + `PUPPETEER_EXECUTABLE_PATH` | `badge-generation.service.ts` | `FOUND_IN_CODE` ⇒ image avec Chromium obligatoire |
| WebSocket Socket.IO (sticky session ou Redis adapter requis pour scale-out) | `websocket.gateway.ts` | `INFERRED_FROM_CODE` |
| Bull Board sur `/admin/queues` | `bull-board.module.ts` | `FOUND_IN_CODE` |
| Backup OVH actuel | `attendee-ems-back/backups/*.dump` | `FOUND_IN_CODE` |
| Deploy scripts shell `deploy.sh`, `deploy-back.sh` | racine module back | `FOUND_IN_CODE` (à relire si pertinent) |
| Nginx config | `attendee-ems-back/nginx/` | `FOUND_IN_CODE` (à relire) |

## 2. Inconnu (ou nécessite confirmation humaine)

- Cloud Run vs GKE vs Compute Engine pour l'API.
- Workers BullMQ : **mêmes containers que l'API** (actuellement le cas, in-process) **ou containers séparés** (recommandé pour Puppeteer/imports lourds).
- Cloud Run scale-to-zero **inacceptable pour des workers BullMQ classiques** (pas de wakeup sur message Redis). Plusieurs options à explorer (mais hors scope ici) : Cloud Run min-instances=1, GKE, Compute Engine.
- Memorystore Redis (managé GCP) vs Redis self-hosted.
- Cloud SQL Postgres vs Postgres self-hosted (currently OVH).
- Cloud Storage GCS vs conservation de R2 (R2 reste accessible depuis GCP, pas d'egress GCP→R2 sortant).
- Scaling cible (req/s, nb concurrents, nb prints/s, taille max imports).
- Budget.
- SLA visés.
- Stratégie multi-region ou single-region (`europe-west1` ?).
- Domaines / TLS (Cloud Load Balancer vs Nginx en frontend).
- Stratégie de secrets (Secret Manager vs env vars).

## 3. Hypothèses raisonnables (à valider, ne pas encore implémenter)

| Élément | Hypothèse |
|---|---|
| Région GCP | `europe-west1` (Belgique) ou `europe-west9` (Paris) |
| API | Cloud Run avec min-instances=1, CPU always allocated pour gérer WebSocket persistant |
| Workers BullMQ | Service Cloud Run distinct avec min-instances=1 et concurrency=1 (Puppeteer) |
| Postgres | Cloud SQL Postgres 16, HA, daily backup, PITR 7 j |
| Redis | Memorystore Redis Standard tier (HA) |
| Storage | Conserver R2 (économies egress, déjà intégré). Réévaluer GCS si besoin de Direct Connect avec d'autres services GCP |
| Logs | Cloud Logging (sink Sentry) |
| Secrets | Secret Manager + injection via env Cloud Run |
| CDN / front | Cloud CDN devant le front statique ; éventuellement Cloud Run pour SSR |
| Sticky session WS | Soit Cloud Run avec session affinity (limitée), soit Redis adapter Socket.IO obligatoire |

## 4. Questions à poser à l'humain ou à l'équipe infra

> Plusieurs sont reportées dans [12-open-questions-for-human.md](12-open-questions-for-human.md).

1. **API** : Cloud Run, GKE Autopilot, ou Compute Engine ?
2. **Workers BullMQ** : in-process avec l'API ou containers séparés ? Combien de pools (un par domaine `imports`, `exports`, `badges`, `prints`, `emails`) ?
3. **Cloud Run scale-to-zero** acceptable ou non pour les workers ? (recommandation : **non**, min-instances ≥ 1 pour drain Redis sans latence)
4. **Long-running workers** : un job d'import 1h est-il dans le périmètre ? Si oui, Cloud Run a un timeout 60 min max (Gen 2). Au-delà : Compute Engine ou Cloud Run Jobs (batch one-shot).
5. **Memorystore Redis** Standard ou Basic tier ? Quel HA RPO acceptable ?
6. **Cloud SQL** : Postgres 16 ? HA cross-zone ? PITR combien de jours ?
7. **Storage** : conserver R2 ou migrer vers Cloud Storage GCS ?
8. **Region** : europe-west1 / west9 / autre ?
9. **Multi-region** : nécessaire ? (probablement non au démarrage)
10. **Sticky sessions / Socket.IO** : adopter `@socket.io/redis-adapter` avant ou pendant la migration ?
11. **Bull Board** : exposer en interne (`/admin/queues` derrière IAP / VPN) ou ajouter un Guard SUPER_ADMIN ?
12. **Budget mensuel cible** ?
13. **Migration data** : Cloud SQL Database Migration Service depuis OVH Postgres ou dump/restore manuel ?
14. **Print Client** : reste sur réseau client (Electron LAN), donc Cloud Run public exposé. Stratégie de whitelist / TLS / token long-lived pour Print Client ?

## 5. Compatibilités à préserver

- **R2 est S3-compatible** ⇒ swap vers GCS demanderait `@google-cloud/storage` (ou conserver `@aws-sdk/client-s3` avec endpoint GCS HMAC). Pas de blocker.
- **BullMQ** dépend de Redis ⇒ Memorystore compatible (Redis 7).
- **Puppeteer** : nécessite Chromium dans l'image worker. Sur Cloud Run, packager Chromium dans le Dockerfile. Augmenter mémoire (1-2 GB minimum).
- **Sentry** : utilisable tel quel sur n'importe quelle plateforme.
- **Socket.IO** : nécessite WebSocket support (Cloud Run le supporte avec timeout 60 min par connexion).

## 6. Risques de migration spécifiques aux jobs

- **Cloud Run scale-to-zero** d'un worker = ne consomme aucun message Redis pendant qu'il dort. ⇒ tous les jobs s'accumulent jusqu'au premier trigger HTTP. Pour BullMQ, on **doit** min-instances ≥ 1.
- **Cloud Run request timeout 60 min Gen 2** : exports/imports > 1 h doivent être splittés ou migrés sur Cloud Run **Jobs** (batch) ou Compute Engine.
- **Connection pool Postgres** : si N workers × M concurrency, dimensionner Cloud SQL ou utiliser PgBouncer.
- **Memorystore** : pas d'authentification AUTH disponible (selon tier) — utiliser VPC privé.
