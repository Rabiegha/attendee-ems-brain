# Staging load-test — Runbook (VPS OVH, stack isolée)

> But : monter une stack **staging iso-prod** sur le **même VPS** que la prod, isolée,
> pour lancer des load tests k6 **sans toucher la prod**. Données = dump prod restauré
> puis **anonymisé**. Emails capturés par **Mailpit** (aucun envoi réel).
>
> Fichiers : [docker-compose.staging.yml](../../docker-compose.staging.yml),
> [.env.staging.example](../../.env.staging.example).

---

## ⚠️ À lire avant de lancer

- **Mesures = plancher prudent**, pas capacité max : staging partage CPU/RAM/IO avec la prod.
  → Lancer en **fenêtre creuse**, surveiller `docker stats` sur la prod pendant chaque run.
  → Les `cpus`/`mem_limit` du compose staging protègent la prod : **ne pas les retirer**.
- **PII** : le dump contient des données réelles. Stack **bindée 127.0.0.1** (jamais publique),
  **anonymiser** après restore (étape 4), **Mailpit** obligatoire (aucun SMTP sortant).
- **Ne pas réutiliser les creds prod** : créer des mots de passe staging dédiés.

---

## 1. Transférer les fichiers sur le VPS

Depuis le repo local :

```bash
cd attendee-ems-back
scp docker-compose.staging.yml .env.staging.example \
    debian@51.75.252.74:/opt/ems-attendee/backend/
```

Le dump est déjà sur le serveur : `/opt/ems-attendee/backups/ems-prod-20260624-234323-pre-pgbouncer.dump`.

## 2. Préparer `.env.staging` sur le VPS

```bash
ssh debian@51.75.252.74
cd /opt/ems-attendee/backend
cp .env.staging.example .env.staging
# Éditer .env.staging : remplir STAGING_USER / STAGING_PWD / STAGING_DB,
# REDIS_PASSWORD, et générer les JWT (openssl rand -base64 64).
# Laisser SMTP_HOST=mailpit et EMAIL_ENABLED=true (capture Mailpit).
```

## 3. Démarrer l'infra staging (sans l'API d'abord)

```bash
docker compose -f docker-compose.staging.yml --env-file .env.staging \
  up -d postgres pgbouncer redis mailpit
# Attendre que postgres soit healthy :
docker compose -f docker-compose.staging.yml ps
```

## 4. Restaurer le dump prod dans le Postgres staging

Le dump a été pris avec un autre rôle/DB (`ems_prod` / `ems_production`). On restaure dans
la DB staging (`STAGING_DB`) en remappant le propriétaire.

```bash
# Copier le dump dans le conteneur staging
docker cp /opt/ems-attendee/backups/ems-prod-20260624-234323-pre-pgbouncer.dump \
  ems-staging-postgres:/tmp/prod.dump

# Restaurer (no-owner / no-privileges => tout appartient à STAGING_USER)
docker exec ems-staging-postgres pg_restore \
  -U STAGING_USER -d STAGING_DB \
  --no-owner --no-privileges --clean --if-exists \
  /tmp/prod.dump

# Nettoyer le dump du conteneur
docker exec ems-staging-postgres rm -f /tmp/prod.dump
```

> Si la restauration renvoie des warnings sur des rôles inexistants (`ems_prod`), c'est
> normal avec `--no-owner` — les objets sont réassignés à `STAGING_USER`.

## 5. ANONYMISER les PII (obligatoire avant tout test email)

```bash
docker exec -i ems-staging-postgres psql -U STAGING_USER -d STAGING_DB <<'SQL'
-- Emails : rendre non-délivrables (domaine réservé RFC 2606 .invalid)
UPDATE attendees       SET email = 'user+' || id || '@staging.invalid';
UPDATE registrations   SET snapshot_email = 'user+' || id || '@staging.invalid'
                       WHERE snapshot_email IS NOT NULL;
-- Téléphones / noms : neutraliser (adapter aux colonnes réelles si besoin)
UPDATE attendees       SET phone = NULL WHERE phone IS NOT NULL;
UPDATE registrations   SET snapshot_phone = NULL WHERE snapshot_phone IS NOT NULL;
SQL
```

> ⚠️ Vérifier les noms de tables/colonnes réels avant d'exécuter (le schéma fait foi).
> Même avec Mailpit, on anonymise : défense en profondeur si un envoi fuyait hors sandbox.

## 6. Démarrer l'API staging

```bash
docker compose -f docker-compose.staging.yml --env-file .env.staging up -d --build api
docker compose -f docker-compose.staging.yml logs -f api   # vérifier le boot
```

## 7. Accès depuis ton poste (tunnels SSH — rien n'est exposé publiquement)

```bash
# API staging -> http://127.0.0.1:3001
ssh -N -L 3001:127.0.0.1:3001 debian@51.75.252.74 &
# Mailpit UI  -> http://127.0.0.1:8025
ssh -N -L 8025:127.0.0.1:8025 debian@51.75.252.74 &
```

Smoke test :

```bash
curl -s http://127.0.0.1:3001/api/health
```

## 8. Lancer les load tests k6

Cibler `http://127.0.0.1:3001` (depuis ton poste via tunnel) **ou** lancer k6 directement
sur le VPS contre le réseau staging. Scénarios = T2/T3/T5 (inscription) du
[plan de charge LFD 2026](lfd-2026-load-test-plan.md).

Pendant chaque run, sur le VPS :

```bash
docker stats   # surveiller que la PROD (ems-api, ems-postgres) reste saine
```

## 9. Teardown

```bash
docker compose -f docker-compose.staging.yml down            # stoppe la stack
docker compose -f docker-compose.staging.yml down -v         # + supprime les volumes (PII)
```

> Après les campagnes, faire `down -v` pour **supprimer les données anonymisées** et ne pas
> laisser traîner de copie de la base sur le VPS.

---

## Checklist de sécurité (avant de lancer un test)

- [ ] `.env.staging` rempli avec des creds **dédiés** (pas ceux de prod).
- [ ] `SMTP_HOST=mailpit`, `EMAIL_ENABLED=true` — vérifié dans Mailpit que les mails arrivent là.
- [ ] R2 vide ou bucket **staging** séparé (pas le bucket prod).
- [ ] Tous les ports staging bindés `127.0.0.1` (cf. compose) — aucun accès public.
- [ ] PII anonymisées (étape 5) avant tout test impliquant l'email.
- [ ] `docker stats` ouvert sur la prod pendant le run ; arrêt si la prod souffre.
