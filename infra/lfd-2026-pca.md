# LFD 2026 — Plan de Continuité d'Activité (PCA)

> Document interne — Réponse technique cahier des charges client.
> Évènement diplomatique public — 4 et 5 septembre 2026.
>
> 🔗 **Version de travail détaillée** (workstream) : [plan-continuite-activite.md](../workstreams/en-cours/infra-scaling-pca/plan-continuite-activite.md).
> Ce fichier-ci = version **synthétique orientée client** ; l'autre = version **longue de travail**.

---

## 1. Objectif

Garantir la continuité du service d'inscription, de check‑in et d'impression badges sur la période
critique J‑15 → J+1, en encadrant les procédures de réponse à incident, les modes dégradés
acceptables et la communication client.

## 2. Périmètre

- Plateforme backend `attendee-ems-back` (NestJS + Prisma + Postgres + Redis + Socket.IO).
- Frontend public `attendee-ems-front` (landing inscription).
- App mobile de check‑in `attendee-ems-mobile`.
- Print client `attendee-ems-print-client`.
- Infrastructure d'hébergement (cf. options A/B du capacity planning).
- Dépendances : SMTP, R2/S3, DNS, Cloudflare.

## 3. Période critique

| Phase | Dates | Astreinte |
|---|---|---|
| Pré‑évènement long | 20 août → 1 sept 2026 | HO + on‑call soir |
| Tension | 2 et 3 sept 2026 | 08 h → 23 h |
| **Jour J** | **4 et 5 sept 2026** | **24/7, 2 ingénieurs** |
| Post‑évènement | 6 sept 2026 | 09 h → 18 h |

## 4. Rôles & responsabilités

| Rôle | Responsabilité | Contact (à renseigner) |
|---|---|---|
| Incident Commander (IC) | Pilote la résolution, communique | _à renseigner_ |
| Tech Lead Backend | Diagnose et applique correctifs API/DB | _à renseigner_ |
| Tech Lead Frontend | Diagnose et patch landing/mobile | _à renseigner_ |
| Ops / DevOps | Scaling, déploiement, restore | _à renseigner_ |
| Référent client | Point de contact client unique | _à renseigner_ |
| Astreinte N1 | Première prise d'appel < 15 min | _à renseigner_ |
| Astreinte N2 (escalade) | Tech Lead disponible < 30 min | _à renseigner_ |

> **Information à confirmer** : liste des contacts d'astreinte + roulement + numéro d'urgence
> publié au client (un seul numéro, pas une liste).

## 5. RTO / RPO

| Composant | RTO (option A) | RTO (option B) | RPO (option A) | RPO (option B) |
|---|---|---|---|---|
| API NestJS | 15 min | 5 min | n/a (stateless) | n/a |
| Base PostgreSQL | 30 min | 10 min | 1 h (dump + WAL local) | 5 min (PITR) |
| Redis (queues) | 10 min | 5 min | 1 min (AOF) | 1 min |
| Stockage R2 | n/a (managé) | n/a | n/a | n/a |

## 6. Scénarios d'incident & procédures

### 6.1 Backend indisponible

- **Détection** : sondes Better Uptime 2 KO consécutifs, 5xx Sentry, Slack alert.
- **Diagnostic** : `docker compose ps`, `journalctl -u …`, logs Nginx.
- **Réponse** :
  1. Vérifier `/api/health/ready` localement sur l'hôte.
  2. Si crash répété : rollback image précédente.
  3. Si DB ou Redis injoignable depuis l'API : voir 6.2 / 6.3.
  4. Si OOM : augmenter `mem_limit` ou scaler.
- **Mode dégradé** : page de maintenance Nginx (`maintenance.html`) renvoyée en 503 avec message
  "Inscriptions temporairement suspendues — réessayez dans quelques minutes". Les utilisateurs en
  cours de remplissage ne perdent rien si le frontend stocke le brouillon en `localStorage`.

### 6.2 Base de données indisponible ou lente

- **Détection** : `/health/ready` KO, alertes Prometheus connexions DB.
- **Réponse immédiate** : passer l'API en **mode lecture seule** (feature flag + middleware
  bloquant POST/PATCH/DELETE sauf check‑in). Annoncer "Inscriptions fermées temporairement".
- **Restore** : si corruption, `pg_restore` du snapshot le plus récent + rejouer les inscriptions
  reçues depuis le snapshot via les logs API (export Loki + replay manuel par script).
- **Mode dégradé** : check‑in continue de fonctionner si la copie locale mobile contient déjà la
  liste (cf. 6.9).

### 6.3 Envoi email indisponible

- **Détection** : taux d'erreur `EmailService` > 5 %, queue email bloquée.
- **Réponse** :
  1. Vérifier crédits / quota du provider.
  2. Bascule sur provider secondaire (à provisionner avant J‑15 — SES ou Mailjet).
  3. Si rien ne fonctionne : laisser les jobs en queue (BullMQ), prolonger TTL à 24 h.
- **Mode dégradé** : la page de confirmation affiche un **lien direct de téléchargement du badge
  PDF** (URL signée R2). L'email partira plus tard. Bandeau d'information sur la landing.

### 6.4 Génération PDF lente ou en échec

- **Détection** : backlog BullMQ > 500 jobs, p95 > 30 s.
- **Réponse** :
  1. Scaler le worker badge (replica +1).
  2. Activer le **template de badge dégradé** (PDF minimaliste, sans photo, sans fond, QR + nom).
  3. Si Puppeteer crash : restart worker, vérifier mémoire Chromium.
- **Mode dégradé** : badge texte simple + QR généré en SVG côté frontend (pas de PDF).

### 6.5 Stockage fichiers indisponible (R2)

- **Détection** : 5xx sur upload, alertes Cloudflare.
- **Réponse** : stocker temporairement les badges en local sur le worker (`/tmp/badges/`) avec un
  flag `pending_upload=true`. Cron de re‑upload toutes les minutes.
- **Mode dégradé** : email contient une pièce jointe inline au lieu d'un lien.

### 6.6 Pic de trafic / saturation CPU/RAM

- **Détection** : CPU > 80 %, file d'attente Node, p95 > 2 s.
- **Réponse** :
  1. Scaler API + worker (autoscaling si option B, manuel si option A).
  2. Activer un **rate limit plus agressif** sur Cloudflare (5 req/min/IP).
  3. Activer file d'attente virtuelle (`Queue-it`‑like) ou page "vous êtes le n°X" via Cloudflare
     Waiting Room si option B.
- **Mode dégradé** : couper le traitement asynchrone non‑critique (notifs admin, exports).

### 6.7 Bot / inscriptions automatisées

- **Détection** : pics anormaux sur une IP/ASN, captcha bypassed, emails jetables détectés.
- **Réponse** :
  1. Activer Cloudflare Bot Fight Mode.
  2. Bloquer IPs/ASN suspects.
  3. Forcer captcha sur toutes les requêtes.
  4. Audit base : `DELETE` soft des registrations frauduleuses (script `scripts/cleanup-fraud.ts` à
     préparer).
- **Mode dégradé** : challenge JavaScript Cloudflare interstitiel sur la landing.

### 6.8 Surbooking sur une jauge

- **Détection** : query SQL `count(registrations) > event.capacity` en sortie d'un cron toutes les
  minutes.
- **Réponse** :
  1. Identifier le créneau, fermer la jauge (set `status=closed`).
  2. Trier les registrations par `created_at` : conserver les N premières en `confirmed`, basculer
     le surplus en `waitlist` + email "vous êtes en liste d'attente" automatisé.
- **Mode dégradé** : annonce client immédiate, ouverture éventuelle d'un créneau supplémentaire.

### 6.9 App de scan en mode hors‑ligne

- **Détection** : perte WebSocket / 5xx persistants côté mobile.
- **Réponse** : l'app mobile bascule automatiquement en mode offline (cache local des
  registrations), enregistre les scans localement, synchronise à la reconnexion.
- **Mode dégradé** : risque double check‑in tant que pas synchro — accepté, traitement en lot post‑
  événement.

> **Information à confirmer** : implémentation effective du mode offline côté `attendee-ems-mobile`.
> Si absent, prévoir un export CSV des inscrits + scanner physique de secours (Honeywell, app tiers).

### 6.10 Print client indisponible

- **Détection** : `printers:updated` ne reçoit plus d'event depuis > 5 min sur un device.
- **Réponse** :
  1. Re‑connecter le print client Electron.
  2. Vérifier alimentation imprimante + driver.
- **Mode dégradé** : impression manuelle depuis le poste check‑in (PDF téléchargé) ; badge papier
  pré‑imprimé pré‑évènement pour les VIP.

### 6.11 WebSocket indisponible

- **Détection** : sonde WS, dashboards live figés.
- **Réponse** : restart gateway, vérifier adapter Redis (option B), bascule polling AJAX (5 s) côté
  dashboard (fallback à coder côté front, à confirmer).
- **Mode dégradé** : dashboard rafraîchi manuellement, alertes par email/Slack à la place des push.

### 6.12 Déploiement raté

- **Détection** : sondes KO post‑deploy, erreurs Sentry nouvelles.
- **Réponse immédiate** : `docker compose up -d --no-deps api` avec le tag précédent. Délai cible
  < 5 min.
- **Mode dégradé** : aucune indisponibilité visible si rollback < 1 min (option B Kubernetes
  rolling).

### 6.13 Incident sécurité / fuite de données

- **Détection** : alerte Sentry, IDS Cloudflare, signalement externe.
- **Réponse** :
  1. Isoler l'instance compromise (couper trafic), snapshot disque pour forensique.
  2. Rotation immédiate des secrets (JWT, SMTP, DB password).
  3. Notification client + DPO + CNIL si applicable < 72 h (RGPD art. 33).
  4. Post‑mortem public sous 7 j.
- **Mode dégradé** : passage en mode lecture seule jusqu'à investigation.

## 7. Procédure backup / restore (mémo)

```bash
# Backup quotidien (cron)
pg_dump -Fc -h $DB_HOST -U $DB_USER ems_production \
  | openssl enc -aes-256-cbc -salt -pass env:BACKUP_PASS \
  > /backups/ems_$(date +%Y%m%d_%H%M).dump.enc
# Upload R2
rclone copy /backups/ems_*.dump.enc r2:ems-backups/

# Restore (en cas d'incident)
openssl enc -d -aes-256-cbc -pass env:BACKUP_PASS \
  -in ems_YYYYMMDD_HHMM.dump.enc \
  | pg_restore -h $DB_HOST -U $DB_USER -d ems_recovery --clean --if-exists
```

> **Action obligatoire** : exécuter la procédure de restore complète sur un environnement de
> recovery au plus tard à **J‑20**, mesurer la durée, joindre le rapport au PCA validé.

## 8. Communication client en cas d'incident

| Sévérité | Délai première notification | Canal | Suivi |
|---|---|---|---|
| P1 (indispo > 5 min) | < 15 min | WhatsApp + email IC client | toutes les 30 min |
| P2 (dégradation perçue) | < 1 h | Email | toutes les 2 h |
| P3 (interne, sans impact) | Post‑mortem hebdo | Slack partagé | hebdo |

Template court P1 :
```
[INCIDENT P1 - LFD 2026] Indisponibilité partielle détectée à HH:MM.
Composant: <API|DB|Email>. Impact: <description>. Action en cours: <rollback|scale|restore>.
Prochaine mise à jour dans 30 min. — IC: <prénom>
```

## 9. Checklists

### 9.1 Avant ouverture des inscriptions (J‑15 → J‑7)

- [ ] Test de charge 3 000 simultanés exécuté, rapport remis au client.
- [ ] Restore DB testé sur env de recovery, durée mesurée < 30 min.
- [ ] Backups automatiques validés (3 derniers présents et chiffrés).
- [ ] Sondes externes configurées (3 régions).
- [ ] Alertes PagerDuty testées (déclenchement volontaire).
- [ ] Astreinte planning publié, contacts à jour.
- [ ] Provider SMTP secondaire provisionné et testé.
- [ ] Captcha activé sur `/register`.
- [ ] Rate limit global configuré (60 req/min/IP, 200 req/s global).
- [ ] Helmet ajouté côté NestJS.
- [ ] Génération badge + email passés en queue BullMQ.
- [ ] Verrou transactionnel sur capacité validé par test concurrentiel.
- [ ] Index DB vérifiés (EXPLAIN sur requêtes critiques).
- [ ] Page de maintenance Nginx prête.
- [ ] Page "vous êtes en file d'attente" prête (option B).
- [ ] Communication client publiée (URL, contact astreinte).
- [ ] Runbook imprimé et accessible offline.

### 9.2 Jour J (4 et 5 septembre 2026)

- [ ] Standup astreinte 07:30 — vérification health, queues, espace disque.
- [ ] Vérification capacités Postgres / Redis / R2 (free space).
- [ ] Vérification cert TLS valide > 30 j.
- [ ] Test smoke complet : visite landing → inscription test → réception email → check‑in test.
- [ ] Snapshot DB manuel à 08:00 et 13:00.
- [ ] Surveillance active 09:00, 11:00, 14:00 (pics ouverture).
- [ ] Standup astreinte 18:00 — debrief journée.
- [ ] Pas de déploiement applicatif sauf hotfix validé par IC + client.

### 9.3 Post‑évènement (J+1 → J+7)

- [ ] Snapshot DB final.
- [ ] Export complet inscriptions + scans (CSV chiffré, livré au client).
- [ ] Désactivation alertes P3 superflues.
- [ ] Rapport SLA chiffré (% dispo, incidents, MTTR, queues).
- [ ] Post‑mortems consolidés.
- [ ] Retour d'expérience client + équipe.
- [ ] Archive logs 90 j (puis purge RGPD).
- [ ] Désactivation captcha agressif et waiting room.
