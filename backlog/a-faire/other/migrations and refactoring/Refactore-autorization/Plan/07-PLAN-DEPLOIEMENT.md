# Plan de déploiement — Refactor Authorization v1

> **Le document maître à suivre le jour J.**
> **Préparé par :** équipe back + front + mobile
> **Validé par :** Clément
> **Date prévue :** TBD (après tous les tests verts)

---

## Vue d'ensemble

Trois deploys à coordonner :

| Ordre | Deploy | Risque | Rollback |
|---|---|---|---|
| 1 | **Back** (Palier 3a + 3b + 3c) | Moyen — migration DB | Restore dump + image précédente |
| 2 | **Front** (refactor v1) | Faible — pure UI | Image Docker précédente |
| 3 | **Mobile** (refactor v1) | Faible — OTA update | Rollback OTA |
| 4 | **Back Palier 4** | Moyen — breaking HTTP | Image précédente (pas de DB) |

⚠️ **Ne PAS faire les 4 deploys le même jour.** Espacer pour observer.

---

## Phase 0 — Préparation (J-3)

### 0.1 — Vérifications préalables

- [ ] Toutes les PRs mergées sur leur branche respective
- [ ] CI verte sur les 3 repos
- [ ] Tous les tests E2E verts (`04-TESTS-E2E-INTEGRATION.md` matrice complète)
- [ ] Smoke test manuel OK sur les 3 rôles (root, admin, agent) — web ET mobile
- [ ] Backup automatique prod confirmé fonctionnel (vérifier la date du dernier dump GCS)
- [ ] Fenêtre de maintenance choisie : **heures creuses** (ex: dimanche 08h00 UTC)
- [ ] Communication client envoyée si applicable

### 0.2 — Pré-flight checklist back

- [ ] Migrations Prisma testées sur copie restaurée du dump prod le plus récent
- [ ] Script `migrate-roots-to-user-system-access.ts` testé sur copie prod (idempotent vérifié)
- [ ] Test du déploiement complet sur staging (si dispo)
- [ ] Image Docker buildée et taguée
- [ ] Tag Git posé sur le commit à déployer (`v1.X-authz-refactor`)

### 0.3 — Pré-flight checklist front

- [ ] Build prod testé localement
- [ ] Variables d'env staging vérifiées
- [ ] Tag Git posé

### 0.4 — Pré-flight checklist mobile

- [ ] Build EAS preview testé sur iOS et Android
- [ ] Décision : OTA update (recommandé si pas de breaking native) ou nouveau build store
- [ ] Tag Git posé

---

## Phase 1 — Deploy BACK (J0)

### 1.1 — Backup obligatoire

```bash
# 1. Snapshot Cloud SQL (instantané, géré par GCP)
gcloud sql backups create --instance=ems-prod-db --description="pre-authz-refactor-$(date +%Y%m%d)"

# 2. Dump explicite supplémentaire
gcloud sql export sql ems-prod-db gs://ems-backups/manual/pre-authz-refactor-$(date +%Y%m%d-%H%M%S).sql --database=ems

# 3. Vérifier que le dump est lisible
gsutil ls -l gs://ems-backups/manual/ | tail -5
```

**Critère :** dump présent, taille > 100 MB, créé dans les 10 dernières minutes.

### 1.2 — Annonce passage en maintenance

- [ ] Bandeau "maintenance en cours" affiché côté front prod (si configuré)
- [ ] Status page mise à jour
- [ ] Slack interne notifié

### 1.3 — Déploiement back

```bash
# 1. Pull image Docker taguée sur le serveur prod (ou trigger Cloud Run)
gcloud run deploy ems-api \
  --image=gcr.io/PROJECT/ems-api:v1.X-authz-refactor \
  --region=europe-west1 \
  --no-traffic   # déployé sans recevoir de trafic encore

# 2. Migration DB depuis la nouvelle révision (qui contient prisma migrate)
gcloud run jobs execute ems-migrate --region=europe-west1
# OU si pas de Cloud Run Job dédié : exec dans le container
gcloud run services proxy ems-api && \
  docker exec <container> npx prisma migrate deploy
```

**Migrations attendues :**
- `20260424120000_palier3a_drop_platform`
- `20260424130000_palier3b_rename_user_roles`
- (toute migration ajoutée entre temps)

### 1.4 — Migration des données root (idempotent)

Le script `migrate-roots-to-user-system-access.ts` (dans `scripts/legacy/migrate-roots-to-user-system-access.ts.done`) doit être rejoué **après** la migration DB pour migrer tout root historique vers `UserSystemAccess`.

```bash
# Exec dans le container API
docker exec ems_api npx ts-node scripts/legacy/migrate-roots-to-user-system-access.ts.done
```

⚠️ **Subtilité :** ce script lit `PlatformUserRole`. Or Palier 3a a DROP cette table.
**Solution :** soit
- (a) le rejouer **AVANT** la migration de Palier 3a (impossible si on déploie tout d'un coup)
- (b) **vérifier qu'il a déjà été joué en local lors de Palier 1** et que tous les roots sont déjà dans `UserSystemAccess` du dump prod
- (c) écrire un nouveau script qui s'appuie sur une trace alternative (ex: dump de `platform_user_roles` pris avant le drop)

**Recommandation :** option **(b)**. Vérifier en local que :
```sql
SELECT COUNT(*) FROM user_system_access WHERE is_root = true;
-- Doit matcher le nombre de roots historiques attendus (rgharghar@choyou.fr + autres)
```

Si le compte est OK en local après restore de dump prod → on est tranquille, pas besoin de rejouer le script en prod.

Sinon → restorer un dump prod **antérieur** à Palier 3a, jouer le script, re-dumper, puis appliquer les migrations.

### 1.5 — Bascule du trafic

```bash
gcloud run services update-traffic ems-api --to-latest --region=europe-west1
```

### 1.6 — Smoke tests prod immédiats

```bash
# Health check
curl https://api.choyou.fr/health
# Login root
TOKEN=$(curl -s -X POST https://api.choyou.fr/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"rgharghar@choyou.fr","password":"***"}' | jq -r .access_token)
# me/orgs
curl -H "Authorization: Bearer $TOKEN" https://api.choyou.fr/auth/me/orgs
# Switch org
# /auth/policy
# /organizations bypass
```

**Critère pour passer à la suite :** tous les endpoints répondent 200 avec les bons payloads.

### 1.7 — Surveillance 30 min

- [ ] Logs Cloud Run : pas d'erreur 500 anormale
- [ ] Sentry : pas de spike d'erreurs back
- [ ] Métriques DB : pas de pic CPU/connexions
- [ ] Latence p95 stable

### 1.8 — Si KO → ROLLBACK back

```bash
# 1. Bascule trafic vers révision précédente
gcloud run services update-traffic ems-api --to-revisions=ems-api-PREV=100 --region=europe-west1

# 2. Restore DB depuis le dump pris à l'étape 1.1
gcloud sql import sql ems-prod-db gs://ems-backups/manual/pre-authz-refactor-XXX.sql --database=ems

# 3. Communication interne + post-mortem
```

---

## Phase 2 — Deploy FRONT (J0 + 1 à 7)

**Pré-requis :** Phase 1 stable depuis au moins 24h, aucune erreur prod.

### 2.1 — Build et deploy

```bash
cd attendee-ems-front
npm run build
# Selon ton infra : Cloud Run / Cloud Storage + CDN / Vercel / Netlify
gcloud run deploy ems-front --image=gcr.io/PROJECT/ems-front:v1.X --region=europe-west1
# OU
gsutil -m rsync -r dist gs://ems-front-prod/
```

### 2.2 — Smoke test front prod

Reprendre la liste manuelle de `04-TESTS-E2E-INTEGRATION.md` section A.

### 2.3 — Surveillance

- [ ] Sentry front : pas de spike
- [ ] Analytics : taux de login/session normal
- [ ] Pas de remontée user

### 2.4 — Si KO → ROLLBACK front

Image précédente, immédiat. Pas de DB à toucher.

---

## Phase 3 — Deploy MOBILE (J0 + 3 à 14)

**Pré-requis :** Phases 1 et 2 stables depuis au moins 48h.

### 3.1 — Choix du mode

- **OTA update** (recommandé si aucun changement natif) → instantané
- **Nouveau build store** → 1 à 7 jours, communiquer aux utilisateurs

### 3.2 — Deploy OTA

```bash
cd attendee-ems-mobile
eas update --branch production --message "Refactor authz v1"
```

### 3.3 — Smoke test mobile prod

Reprendre `04-TESTS-E2E-INTEGRATION.md` section B sur device réel iOS + Android.

### 3.4 — Surveillance

- [ ] Sentry mobile : pas de spike
- [ ] Crash-free rate stable
- [ ] Pas de remontée user

### 3.5 — Si KO → ROLLBACK mobile

```bash
eas update --branch production --message "Rollback to previous" --republish <previous-update-id>
```

---

## Phase 4 — Deploy BACK Palier 4 (J0 + 14 minimum)

**Pré-requis :**

- [ ] Phase 1, 2, 3 stables
- [ ] Au moins 7 jours sans erreur sur les nouveaux endpoints
- [ ] Front prod ne lit plus `mode`/`isPlatform` (vérifié par grep + observation logs)
- [ ] Mobile : > 90% des sessions actives sur la version refactorée (analytics)

Ensuite, déploiement standard (cf `05-PALIER-4-cleanup-http-deprecated.md`).

⚠️ Pas de migration DB → rollback simple si besoin.

---

## Phase 5 — Tests post-prod

Cf `08-TESTS-POST-PROD.md`.

---

## Communication

| Audience | Quand | Quoi |
|---|---|---|
| Équipe interne | J-3 | Annonce planning |
| Équipe interne | J0 -1h | "Maintenance dans 1h" |
| Équipe interne | J0 +30min | "Deploy OK, surveillance en cours" |
| Équipe interne | J0 +24h | "RAS, on continue Phase 2" |
| Clients (si applicable) | J-1 | Email maintenance prévue |
| Clients | J0 +1h | "Maintenance terminée" |

---

## Checklist du jour J

- [ ] Backup DB confirmé (étape 1.1)
- [ ] Migrations testées sur copie dump récente
- [ ] Image Docker back taguée et disponible
- [ ] Image Docker front taguée et disponible
- [ ] EAS update mobile préparé
- [ ] Équipe disponible pendant 4h post-deploy
- [ ] Canal Slack dédié ouvert
- [ ] Document de rollback à portée de main
- [ ] Alertes Sentry/Cloud Monitoring activées

## Ce qu'on ne fait PAS le jour J

- ❌ Pas de feature en parallèle
- ❌ Pas de touche à autre chose qu'à ces 3 repos
- ❌ Pas de deploy front/mobile sans avoir validé le back
- ❌ Pas de Palier 4 dans la même fenêtre que Palier 3
- ❌ Pas de rollback à moitié — soit on rollback tout, soit on fix forward
