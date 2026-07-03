# LFD 2026 — Stratégie SLA 99,9 %

> Document interne — Réponse technique cahier des charges client.
> Évènement diplomatique public — 4 et 5 septembre 2026.
> Source : audit `attendee-ems-back` (branche `gcp-migration-stable`), 22 mai 2026.

---

## 1. Définition du SLA

| Élément | Valeur |
|---|---|
| Disponibilité cible | **99,9 %** |
| Périmètre | API publique d'inscription, API de check‑in, dashboard temps réel |
| Hors périmètre | Outils tiers (SMTP, R2/S3) — couverts par leurs propres SLA |
| Période critique | **J‑15 → J+1**, soit du 20 août 2026 (00:00 Europe/Paris) au 6 septembre 2026 (23:59) |
| Durée de la fenêtre critique | ≈ 408 h |
| Indisponibilité max tolérée | **≈ 24 min sur 17 jours** (99,9 %) |
| Sur J et J+1 uniquement | ≈ 2 min 53 s |

### 1.1 Définition d'une indisponibilité

Est considéré comme indisponible un service pour lequel, sur une fenêtre glissante d'**1 minute**,
**au moins 50 %** des requêtes externes vers les endpoints suivants renvoient un code ≥ 500 ou
ne répondent pas dans la SLO de latence :

| Endpoint | SLO latence | Méthode de mesure |
|---|---|---|
| `GET /api/health` | < 500 ms | Sonde externe (Better Uptime / Uptime Robot) toutes les 30 s |
| `GET /api/public/events/:token` | p95 < 800 ms | Sonde synthétique 60 s |
| `POST /api/public/events/:token/register` | p95 < 1 500 ms | Sonde synthétique 60 s (compte gabarit) |
| `POST /partner-scans` | p95 < 600 ms | Sonde authentifiée |

> Les actions asynchrones (génération badge, envoi email) sont mesurées séparément en
> **taux de complétion** : ≥ 99 % des jobs terminés en < 60 s.

---

## 2. Mesure de la disponibilité

### 2.1 Sondes externes

| Sonde | Fréquence | Localisation | Outil proposé |
|---|---|---|---|
| Health API | 30 s | 3 régions (FR, EU‑W, US‑E) | Better Uptime |
| Smoke inscription end‑to‑end | 5 min | FR | k6 cloud / cron GitHub Actions |
| Latence DB (via API) | 1 min | FR | sonde interne `/health/ready` |
| Cert TLS expiration | 1 j | FR | Better Uptime |

### 2.2 Calcul mensuel & reporting

Disponibilité = `1 − (durée_indispo_minutes / durée_fenêtre_minutes)`.
Reporting hebdomadaire sur la fenêtre critique, envoyé au client tous les vendredis à 17 h.

---

## 3. Health & readiness

À ajouter dans `src/health/` :

| Endpoint | Vérifie | Code 200 si |
|---|---|---|
| `GET /api/health` (existant) | Process up | Toujours |
| `GET /api/health/live` (à ajouter) | Event‑loop responsive | Toujours sauf shutdown |
| `GET /api/health/ready` (à ajouter) | DB + Redis joignables, migrations à jour | Toutes deps OK |

Utilisés par :
- Le LB (readiness) — retire de la rotation si KO.
- Kubernetes / Docker (`healthcheck`, `livenessProbe`, `readinessProbe`).

---

## 4. Monitoring & alerting

### 4.1 Stack proposée

| Couche | Outil | Existant ? |
|---|---|---|
| Errors | Sentry (`src/instrument.ts`) | ✅ |
| Logs | Pino → Loki / Grafana Cloud / Cloud Logging | Pino ✅, agrégation à mettre en place |
| Metrics | Prometheus + Grafana (à ajouter) ou Cloud Monitoring | ❌ |
| Uptime externe | Better Uptime / Uptime Robot | ❌ |
| APM | Sentry Performance ou Datadog APM | partiel via Sentry |

### 4.2 Alertes critiques (PagerDuty / Opsgenie / Slack #ems-incidents)

| Alerte | Seuil | Sévérité | Action |
|---|---|---|---|
| API 5xx rate | > 1 % sur 5 min | P1 | Page astreinte |
| Health check externe KO | 2 sondes consécutives | P1 | Page astreinte |
| p95 latence `/register` | > 2 s sur 5 min | P2 | Slack |
| Queue badge backlog | > 500 jobs en attente > 2 min | P2 | Slack |
| Queue email backlog | > 1 000 jobs ou taux d'échec > 5 % | P2 | Slack |
| CPU API | > 80 % sur 5 min | P3 | Slack |
| Connexions DB | > 80 % pool | P2 | Slack |
| Espace disque DB | > 80 % | P2 | Slack |
| Certificat TLS | < 14 j d'expiration | P3 | Slack |

---

## 5. Déploiement sans interruption

### 5.1 Stratégie

| Période | Stratégie de déploiement |
|---|---|
| Avant J‑15 | Rolling update standard (2 instances, 1 par 1) |
| J‑15 → J‑1 | **Gel des déploiements** sauf hotfix critique validé par astreinte + client |
| J et J+1 | **Gel total** sauf rollback |
| Après J+1 | Reprise normale |

### 5.2 Procédure de rollback

1. Identifier le tag stable précédent (`docker images`, `git tag`).
2. Re‑déployer l'image précédente (`docker compose up -d --no-deps api`).
3. Vérifier `/api/health/ready` + sondes externes.
4. Si migration Prisma incompatible : exécuter `prisma migrate resolve --rolled-back` puis appliquer
   un correctif aller dans la migration suivante (jamais de `migrate reset` en prod).

### 5.3 Migrations DB

- Toujours **rétro‑compatibles** pendant la fenêtre critique : `ADD COLUMN NULL`, jamais de `DROP`
  ou `RENAME` direct.
- Exécutées **hors trafic** (fenêtre 03:00–05:00 Europe/Paris).
- Backup automatique avant migration.

---

## 6. Backups

| Type | Fréquence | Rétention | Test restore |
|---|---|---|---|
| Snapshot DB managé | Auto (Cloud SQL) | 30 j (option B) / 7 j (option A) | 1 fois avant J‑15 |
| `pg_dump` logique | Quotidien 02:00 | 30 j sur R2 chiffré | 1 fois avant J‑15 |
| PITR | Continu (option B) | 7 j | testé J‑20 |
| Code (images Docker) | À chaque tag | 90 j | rollback test J‑20 |

> **Obligation** : un **dry‑run de restore complet sur environnement vierge** doit être exécuté
> au plus tard à **J‑20**, avec validation chronométrée (cible RTO 30 min).

---

## 7. Conditions et limites du SLA

Le SLA 99,9 % ne couvre pas :

- Panne d'un service tiers (SMTP, R2, DNS, Cloudflare, opérateur réseau client).
- Cas de force majeure (panne datacenter régionale, attaque DDoS volumétrique > 10 Gbps non
  filtrée par le WAF inclus).
- Indisponibilité due à un changement applicatif demandé par le client pendant la fenêtre de gel.
- Utilisation hors périmètre (volumétrie > celle dimensionnée dans `lfd-2026-capacity-planning.md`).
- Endpoints de back‑office et exports CSV : SLO 99 %, latence non garantie.

---

## 8. Runbook abrégé (extrait — voir PCA pour le détail)

| Symptôme | Première vérif | Action |
|---|---|---|
| 5xx massifs | Sentry + logs API | Rollback dernier déploiement |
| p95 explose | Grafana CPU + DB | Scale API +1 replica, vérifier queries lentes |
| Queue backlog badge | Redis `bull:badge:wait` | Scale worker‑badge +1, baisser qualité PDF |
| Surbooking détecté | Query SQL sur capacité | Geler ouvertures du créneau, audit registrations |
| SMTP KO | `EmailService` logs | Bascule provider secondaire (à provisionner) |
| WebSocket désync | sondes WS | Restart gateway, vérifier adapter Redis |

---

## 9. Astreinte technique

| Plage | Niveau | Temps de réponse |
|---|---|---|
| J‑15 → J‑3 | Heures ouvrées + astreinte téléphonique 18 h → 22 h | < 1 h |
| J‑2 et J‑1 | Astreinte 08 h → 23 h | < 30 min |
| **J et J+1 (4 et 5 sept)** | **Astreinte 24/7, 2 ingénieurs** | **< 15 min** |
| J+1 → J+15 | Astreinte téléphonique 09 h → 18 h | < 2 h |

Outils : PagerDuty / Opsgenie + Slack #ems-incidents + canal WhatsApp dédié client.

---

## 10. Engagements proposés au client

| Engagement | Valeur |
|---|---|
| SLA disponibilité fenêtre critique | 99,9 % |
| SLA p95 latence `/register` | < 1 500 ms |
| Délai prise en charge incident P1 | < 15 min (24/7 sur J/J+1) |
| Délai résolution ou contournement P1 | < 1 h |
| RTO global | 30 min |
| RPO | 5 min (option B avec PITR) / 1 h (option A) |
| Test de charge contractuel à 3 000 simultanés | Réalisé au plus tard J‑10, rapport remis J‑7 |
| Compte‑rendu d'incident | Sous 48 h après résolution |
