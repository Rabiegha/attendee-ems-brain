# Plan Monitoring & alerting (0-MON) — minimal event-ready

> **Statut :** ACTIF · **Date :** 2026-07-11 · **Chantier :** 0-MON ([00-plan-action §P0](../00-plan-action.md))
> **Périmètre :** `attendee-ems-back` + `attendee-ems-front` + VPS prod.
> **Objectif :** ne plus voler à l'aveugle sur la prod — être **alerté** avant que ça casse
> (surtout **disque plein** = Postgres ne peut plus écrire le WAL = **API down**).
>
> **Compagnons :** [décision — tenir sans GCP §4.3](../K-resilience-event/tenir-event-sans-gcp.md) (disque/WAL) ·
> [ws résilience event](../../../a-faire/resilience-event-lfd2026/README.md).

---

## 0. État réel vérifié (2026-07-11)

| Brique                                               | État                         | Détail                                                                                                                      |
| ---------------------------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Sentry back**                                      | ✅ **Code fait** — à activer | `src/instrument.ts` (`Sentry.init` conditionnel si `SENTRY_DSN`), `@sentry/node` installé, `common/logger/sentry-logger.ts` |
| **Sentry front**                                     | ✅ **Code fait** — à activer | `@sentry/react` + `@sentry/vite-plugin` installés, `src/shared/lib/logger.ts`, upload source maps en prod                   |
| **Uptime + alerte**                                  | ✅ **FAIT** (Rabie)          | monitoring externe sur `/health` déjà en place                                                                              |
| **Healthchecks**                                     | ✅ partiel                   | healthchecks Docker + endpoints `/health` / `/api/health`                                                                   |
| **Métriques système (CPU/RAM/🔴 disque)**            | ❌ **absent**                | aucune métrique temps réel                                                                                                  |
| **Alerting sur seuils** (disque / CPU / file BullMQ) | ❌ **absent**                | rien ne prévient avant saturation                                                                                           |
| `attendee-ems-logs`                                  | ℹ️                           | **visualiseur de logs**, PAS de l'alerting                                                                                  |

> **Conclusion :** l'error-tracking (Sentry) et l'uptime sont **déjà là**. Le **vrai trou** = pas de
> **métriques système** ni d'**alerte seuils** (disque en premier). C'est ça qu'il reste à poser.

---

## 1. Reste à faire (le vrai gap)

### 1.1 Activer Sentry (config, pas du code)

- Créer projet(s) Sentry → récupérer le **DSN** → poser `SENTRY_DSN` (+ `SENTRY_ENVIRONMENT`) en **prod** et **staging**.
- Vérifier qu'une exception remonte bien (back + front). Le code s'active tout seul dès que le DSN est présent.

### 1.2 Métriques CPU / RAM / 🔴 disque → **Netdata**

- **1 container Netdata** sur le VPS → dashboard temps réel + **alertes intégrées** (CPU, RAM, **disque**, I/O).
- Léger, auto-découverte, pas de stack Prometheus+Grafana à monter pour l'event.

### 1.3 Alerting sur seuils (la protection n°1)

- 🔴 **Disque ≥ 70 / 80 / 90 %** → alerte (disque plein = **Postgres WAL bloqué = API down**). Alerte Netdata native.
- **CPU élevé soutenu** (goulot mesuré de l'inscription) → alerte.
- **File BullMQ qui gonfle** → petite **métrique custom** exposée + alerte.
- Canal : Slack/email (réutiliser le webhook de l'uptime si possible).

### 1.4 Garde-fous disque (couplés à l'alerte)

- **Rotation des logs** (`logrotate` / Docker `max-size`).
- **Offload PDF + backups → R2** (pas sur le disque local du VPS).
- Surveiller les **slots de réplication bloqués** (WAL jamais recyclé).

---

## 2. Répartition

| Tâche                                                 | Qui                               |
| ----------------------------------------------------- | --------------------------------- |
| Activer Sentry (DSN prod+staging, vérif événements)   | 👤 **toi** (comptes/secrets)      |
| Service **Netdata** (docker-compose + config alertes) | 🤖 **moi** (code)                 |
| Déployer Netdata sur le **VPS prod**                  | 👤 **toi** (accès VPS)            |
| Alerte **file BullMQ** (métrique custom)              | 🤖 **moi** (code)                 |
| Rotation logs + offload R2 + slots réplication        | 🤖 moi (config) / 👤 toi (deploy) |
| Webhook Slack/email pour les alertes seuils           | 👤 toi                            |

> **Simple / solo ?** Même profil que staging : je fais **tout le code/config** (Netdata, métrique BullMQ,
> rotation), mais l'**activation Sentry (DSN)**, les **comptes** et le **déploiement VPS prod** = tes mains.

---

## 3. Ordre de livraison

| #   | Étape                                                                  | Qui   | Risque |
| --- | ---------------------------------------------------------------------- | ----- | ------ |
| 1   | **Activer Sentry** (DSN) — le code est déjà là                         | 👤    | nul    |
| 2   | **Netdata** (métriques CPU/RAM/disque) + **alerte disque ≥70/80/90 %** | 🤖+👤 | faible |
| 3   | **Alertes CPU + file BullMQ**                                          | 🤖+👤 | faible |
| 4   | **Garde-fous disque** (rotation logs, offload R2)                      | 🤖+👤 | faible |

---

## Références

- [00-plan-action §P0 — 0-MON](../00-plan-action.md) — périmètre minimal décidé.
- [décision — tenir sans GCP §4.3](../K-resilience-event/tenir-event-sans-gcp.md) — disque/WAL, la protection n°1.
- [ws résilience event](../../../a-faire/resilience-event-lfd2026/README.md) — checklist finale avant J-7.
- Code existant : back `src/instrument.ts` · front `src/shared/lib/logger.ts`.
