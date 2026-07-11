# Post-mortem — 502 prod du 25/06 (collision de nom de projet Docker Compose)

> **Statut :** ✅ corrigé (commit `5f72daa`, 25/06 02:19). Post-mortem rédigé le 2026-07-10.
> **App :** back / infra (VPS OVH prod).
> **Impact :** API prod en **502** pendant **quelques minutes** dans la nuit du 25/06,
> **sans impact sur les sessions** (trafic quasi nul à cette heure).
> **Garde-fous :** [garde-fous-deploiement-staging.md](../../workstreams/en-cours/lfd2026/A-I-leviers/garde-fous-deploiement-staging.md)

---

## 1. Résumé

Pendant le montage de la stack **staging** de load-test sur le **même VPS** que la prod
(nuit du 24 au 25/06), un `docker compose up` du fichier staging a été lancé **sans nom de
projet explicite**. Docker Compose a dérivé le nom de projet du **nom du dossier courant**
(`backend`) — c'est-à-dire **le même projet que la prod**. Compose a donc considéré les
conteneurs prod comme faisant partie de « son » projet et les a **arrêtés/recréés** →
API prod indisponible → **502 Bad Gateway** via nginx.

## 2. Cause racine

- Quand ni `name:` (top-level du compose), ni `-p`, ni `COMPOSE_PROJECT_NAME` ne sont
  fournis, **Compose utilise le nom du répertoire courant comme nom de projet**.
- La prod tourne depuis `/opt/ems-attendee/backend/` → projet implicite `backend`.
- Le compose staging a été lancé **depuis ce même dossier** → même projet `backend` →
  Compose a « adopté » les conteneurs prod et réconcilié l'état vers la définition staging
  (arrêt/recréation des services prod).
- **Facteur aggravant :** aucun monitoring/alerting à l'époque (cf. chantier 0-MON) —
  la détection n'a été possible que parce qu'on travaillait dessus au même moment.

## 3. Timeline (25/06)

| Heure         | Événement                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01:08         | Premiers commits de la stack staging (`c7247ae`, pgBouncer, dump/restore…).                                                                        |
| ~01:15–02:15  | Lancement du compose staging sur le VPS **sans nom de projet** → collision avec le projet prod `backend` → conteneurs prod recréés → **502 prod**. |
| ~02:15        | **Détection manuelle** (surveillance active pendant la session de nuit — pas d'alerting).                                                          |
| 02:19         | **Correctif `5f72daa`** : `name: ems-staging` dans `docker-compose.staging.yml` → projets Docker distincts. Prod relancée, `health=200`.           |
| Suite de nuit | Prod surveillée pendant les campagnes k6 : intègre (`health=200`, CPU ≈ 0 %).                                                                      |

## 4. Détection

- **Manuelle** : constat du 502 pendant la session de travail. Durée courte uniquement
  grâce à la présence active au même moment — **coup de chance, pas un process**.
- Ce qui aurait dû exister : uptime externe + alerte sur `/api/health` (chantier **0-MON**
  du [plan d'action LFD 2026](../../workstreams/en-cours/lfd2026/00-plan-action.md)).

## 5. Correctif appliqué

- `name: ems-staging` (top-level) dans `docker-compose.staging.yml` (`5f72daa`) —
  **la** protection structurelle : la collision de nom de projet devient impossible.
- Repris et complété dans la refonte propre (branche `infra/staging-stack`, PR #14) :
  `container_name: ems-staging-*` sur chaque service, réseau dédié `ems-staging-network`,
  ports bindés `127.0.0.1`, limites `cpus`/`mem_limit` pour protéger la prod.

## 6. Garde-fous (pour que ça ne se reproduise pas)

Détail et suivi d'avancement dans
[garde-fous-deploiement-staging.md](../../workstreams/en-cours/lfd2026/A-I-leviers/garde-fous-deploiement-staging.md) :

1. **Isolation compose** (`name:`, `container_name`, réseau dédié, `127.0.0.1`) — ✅ en place.
2. **Répertoire séparé** sur le VPS pour le checkout staging (jamais dans le dossier prod).
3. **Jamais de `docker compose up` « nu »** sur le VPS — toujours `-f` + `--env-file`,
   idéalement via un script `staging-up.sh`.
4. **Checklist avant/après chaque déploiement staging** : `docker compose ls` avant,
   `docker ps` après (seuls des `ems-staging-*` ont bougé), `curl` du health **prod**.
5. **Limites de ressources** conservées dans le compose staging + runs k6 en fenêtre creuse.

## 7. Leçons

- Un déploiement manuel sur un hôte partagé prod/staging est une **erreur humaine qui
  attend son heure** : l'isolation doit être **structurelle** (config), pas disciplinaire.
- Sans monitoring, la durée d'un incident = le hasard de qui regarde. L'alerte uptime
  (2 h de setup) aurait transformé ce quasi-accident en non-événement.

## Références

- Audit détaillé : [journal 2026-06-26 §P3](../../workstreams/en-cours/gcp-migration/infra-scaling-pca/journal/2026-06-26-audit-branche-staging.md)
- Runbook staging : [infra/staging/staging-setup.md](../../infra/staging/staging-setup.md)
- Mention durée/impact : [bug 499 §« Pas lié à l'incident 502 »](../en-cours/2026-07-01-eject-mobile-refresh-rotation-499.md)
