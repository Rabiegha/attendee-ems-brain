# Plan d'action — LFD 2026 (refonte propre + chantiers à venir)

> **Objet :** plan d'action consolidé du workstream pour l'événement La Fabrique de la Diplomatie
> 2026 (MEAE, ~12 400 inscriptions, 4–5 sept 2026). Couvre **(P0) les fondations ultra-urgentes**
> (CI/CD, monitoring — inexistants aujourd'hui), (A) la **refonte propre** du §7 de l'audit, puis
> (B→H) les **nouveaux chantiers** (chaîne email/billet, sécurité QR, sauvegarde DB, continuité,
> wallet V2, **inscriptions par session**). Les durées supposent un travail **assisté par IA**.
> **Date :** 30 juin 2026 · **Rattachement :** workstream `lfd2026`
> **Références :** [audit-branche-staging.md](../../a-faire/infra-scaling-pca/journal/2026-06-26-audit-branche-staging.md) ·
> [diagnostic email/billet/wallet](./diagnostics/email-billet-wallet.md) ·
> [plan-continuite-activite.md](../../a-faire/infra-scaling-pca/plan-continuite-activite.md) ·
> [brief-backup-automatique-db.md](../../../backlog/a-faire/brief-backup-automatique-db.md)

---

## Vue d'ensemble des chantiers

> **Cadre :** échéance **1 mois**, **2 devs**. Périmètre **descopé** pour tenir (cf. §4 + planning).

| # | Chantier | Priorité | Statut | Détail |
|---|---|---|---|---|
| **A** | **Refonte propre (§7 audit)** — rejouer les leviers proprement | 🔴 **Priorité n°1** | À faire | §1–§2 ci-dessous |
| **I** | **⚡ Levier débit n°1 — compteur capacité dénormalisé + transaction courte** (lève le plafond ~30/s) | 🔴 **Haute (perf)** | 🟡 Diagnostiqué, non codé | §3-I |
| **C** | Migration ESP (Brevo/Scaleway) — délivrabilité ~12 400 envois | 🔴 Haute | ⚠️ **warm-up à lancer J1** | §3-C |
| **0-MON** | Monitoring & alerting **minimal** (uptime / Sentry / CPU) | 🔴 Haute | ❌ À créer | §P0 |
| **0-CI** | CI/CD **minimaliste** (build+test PR + deploy staging) | 🔴 Haute | ❌ À créer | §P0 |
| **E** | Sauvegarde DB automatique (cron + offsite + restore testé) | 🔴 Haute | Brief prêt | §3-E |
| **D** | Sécurité QR (signature HMAC + durcir route publique) | 🔸 Moyenne | **Gardé** (simple) | §3-D |
| **B** | Chaîne email → billet PDF (rendu **Gotenberg/Cloud Run**, hors VPS) | 🔴 Haute | ✅ Moteur tranché (D) | §3-B |
| **H** | **Inscriptions par session** (lien public par session + capacité/waitlist + refonte front) | 🔴 Haute (**fonctionnel**) | 🟡 Cadré, prêt à découper | §3-H |
| **F** | Continuité — **HA réplication** | 🔵 **Reporté → migration GCP** | HA/PITR natifs Cloud SQL | §3-F |
| **G** | Wallet Apple + Google | ⚪ V2 | Hors périmètre event | §3-G |

> **⚡ Ordre :** **A (refonte) d'abord** — elle débloque le déploiement propre de tout le reste.
> En parallèle, lancer **dès J1** le warm-up ESP (délai calendaire). **CI/CD et monitoring restent
> volontairement minimalistes** pour tenir le mois. La **HA complète est reportée à la migration GCP**
> (Cloud SQL fournit HA + PITR managés → inutile de la construire à la main maintenant). G = V2.

---

## ⚡ P0 — Fondations minimales (CI/CD + monitoring, version réduite)

> Ces deux chantiers sont **transverses** : ils protègent **tous** les autres. En contexte 1 mois,
> on les garde en **version minimale** — assez pour ne pas voler à l'aveugle sur la prod, sans
> y engloutir le planning, à quelques semaines d'un événement institutionnel MEAE à fort enjeu.

### Chantier 0-CI — CI/CD (minimal)  🔴 Haute

**État réel (vérifié) :** **aucun pipeline.** Pas de `.github/workflows`, pas de GitLab CI, ni
CircleCI/Jenkins. Le déploiement = **scripts shell manuels** (`deploy-back.sh`, `deploy-front.sh`…)
qui font un `git pull` de `main` **directement sur le VPS prod**, rebuild le container et lancent
les migrations Prisma.

**Pourquoi c'est dangereux aujourd'hui :**
- ❌ **Aucun test ni build vérifié avant la prod** → une régression part en prod sans filet.
- ❌ **Build fait sur le VPS** → si le build casse, il casse **en prod**.
- ❌ **Déploiement manuel = erreur humaine** — l'incident **502 du 25/06** (collision de nom de
  projet Docker) est exactement ce type de risque.
- ❌ **Pas de rollback** automatisé, **pas de gating** par PR/review.

**À mettre en place (version MINIMALE pour le mois) :**

| Action | Temps estimé |
| --- | --- |
| **CI** : sur chaque PR → lint + build + tests (back & front), **bloque le merge si rouge** | ~1 jour |
| **Déploiement staging** semi-auto + **healthcheck post-deploy** | ~1 jour |
| Gestion des **secrets** (fin des `.env` copiés à la main) | ~2 h |
| **Sous-total 0-CI (minimal)** | **~2 jours** |
| 🔵 *Reporté post-event :* CD prod complet (déploiement auto + rollback) | ~1-2 j |

> **Décision :** on garde le **déploiement prod manuel** ce mois-ci, mais encadré par une
> **checklist pré-déploiement** (pour éviter un nouveau 502 type collision Docker). Le CD complet
> (auto + rollback) est **reporté après l'event**.

### Chantier 0-MON — Monitoring & alerting (minimal)  🔴 Haute

**État réel (vérifié) :** **quasi inexistant.** Uniquement des **healthchecks Docker** (niveau
conteneur) + endpoints `/health` / `/api/health`. **Aucun** uptime externe, **aucune** métrique
(Prometheus/Grafana), **aucun** error-tracking (Sentry), **aucun** alerting. `attendee-ems-logs`
est un **visualiseur de logs**, pas un système d'alerte.

**Pourquoi c'est dangereux aujourd'hui :**
- ❌ Si l'API tombe **à 3 h du matin** pendant les inscriptions, **personne n'est alerté** → on
  découvre par les plaintes clients.
- ❌ **Aucune visibilité temps réel sur le CPU** — alors que c'est **le goulot mesuré** de l'inscription.
- ❌ **Exceptions applicatives invisibles** (pas de Sentry) → des erreurs passent sous le radar.

**À mettre en place (MVP) :**

| Action | Temps estimé |
| --- | --- |
| **Uptime externe + alerte** (UptimeRobot / Better Uptime) sur `/health` → SMS/Slack si down | ~2 h |
| **Error-tracking** (Sentry) back + front → exceptions remontées | ~demi-journée |
| **Métriques système** (CPU/RAM/disque) + dashboard (Grafana+node-exporter, ou Netdata) | ~1 jour |
| **Alerting sur seuils** (CPU élevé, file BullMQ qui gonfle, disque plein) | ~demi-journée |
| **Sous-total 0-MON** | **~2 jours** |

> **⚡ Ces deux chantiers (~4 jours, version minimale) sont des fondations à poser tôt** (S1), en
> parallèle de la refonte (A). Tant qu'ils ne sont pas là, chaque modif sur la prod se fait
> **sans filet de sécurité**.

---

## 0. Principe directeur (chantier A — refonte)

**Ne rien perdre, tout rejouer proprement.** La branche `staging` reste l'archive de référence
(mesures réelles + code validé). On repart d'une base propre depuis `main` et on rejoue chaque
levier dans son propre commit, diff minimal, mesure avant/après.

> Les durées ci-dessous sont du **temps de travail effectif** (hors temps d'exécution des tests
> de charge k6, qui tournent en tâche de fond).

---

## 1. Détail des étapes + temps estimé

### Étape 0 — Geler & archiver
**Temps : ~10 min**

- `git tag archive/staging-2026-06-25` sur `attendee-ems-back` **et** `attendee-ems-brain`.
- Pousser les tags (`git push origin --tags`) → référence figée, irréversible à perdre.
- Vérifier que la branche `staging` n'est **jamais** ciblée pour un déploiement prod (cf. P4).

**Livrable :** 2 tags d'archive poussés. Filet de sécurité posé.

---

### Étape 1 — Séparer reformatage et logique
**Temps : ~15 min**

- Désactiver le reformatage à la sauvegarde (ou figer la config Prettier du repo).
- Si besoin d'un commit `style: prettier`, le faire **seul**, isolé, pour que les commits de
  logique suivants restent en **diff minimal**.

**Livrable :** environnement de travail qui ne génère plus de bruit de reformatage.

---

### Étape 2 — Rejouer chaque levier (branche + commit propres)

> Le cœur du travail. Un levier = une branche = un commit = une PR = une mesure.
> Les leviers déjà **propres** dans l'archive (L1, L2, L4, L5, L10, L11, L14) sont
> **cherry-pick** tels quels ; seuls les leviers **noyés** sont rejoués à la main.

| Ordre | Levier | Branche | Nature | Temps estimé |
| --- | --- | --- | --- | --- |
| 1 | **L9** transaction allégée (`public.service.ts`) — diff minimal + test de non-régression | `feat/register-tx-slim` | rejeu à la main (~50 lignes utiles) | **~45 min** |
| 2 | **L7** email async BullMQ | `feat/email-async-bullmq` | rejeu guidé (nouveaux fichiers) | **~30 min** |
| 3 | **L8** worker `PROCESS_ROLE` (Voie A) | `feat/process-role-worker` | rejeu à la main (gating `main.ts`) | **~20 min** |
| 4 | **L3** `directUrl` Prisma | `chore/prisma-directurl` | cherry-pick ciblé (+5 lignes) | **~10 min** |
| 5 | **L1/L2/L10** stack staging + pgBouncer + pool | `infra/staging-stack` | cherry-pick (déjà propre) | **~20 min** |

**Sous-total étape 2 : ~2 h 05** (hors temps d'exécution des campagnes k6).

> Chaque PR embarque sa **mesure avant/après** (réutiliser les scripts k6 de L14 déjà propres).
> Les tests de charge tournent en fond et ne comptent pas dans le temps de travail.

---

### Étape 3 — Réconcilier la doc (P2)
**Temps : ~15 min**

- Corriger `infra-scaling-pca/README.md` : remplacer « ~33/s / plafond DB » par les **mesures
  réelles** (~25/s, plafond **CPU**, DB sous-utilisée).
- Les rapports infra font déjà foi (corrigés le 26/06) → simple alignement.

**Livrable :** doc cohérente avec les mesures réelles (une seule version de la vérité).

---

### Étape 4 — Post-mortem incident prod (P3)
**Temps : ~25 min**

- Créer `attendee-ems-brain/bugs/2026-06-25-prod-502-collision-compose.md`.
- Contenu : cause (collision nom de projet Docker `backend`), timeline, détection, correctif
  (`name=ems-staging`), garde-fou (check de pré-déploiement).

**Livrable :** post-mortem exploitable, garde-fou documenté.

---

### Étape 5 — Isoler le compose prod (P4)
**Temps : ~20 min**

- Extraire les modifications historiques de `docker-compose.prod.yml` (+52) dans une **PR dédiée**.
- Ne **pas** fusionner tant que non revue séparément.
- Ne pas confondre avec l'ajout du 29/06 (`3f0a69d`, déjà déployé et validé — cf. §9).

**Livrable :** changements prod isolés, en attente de revue explicite.

---

### Étape 6 — Ce qui reste interdit / en attente
**Temps : ~5 min** (note de cadrage, pas de code)

- **L13 cluster (Voie B)** : reste **interdit en prod** jusqu'à livraison du workstream
  `api-scaling-clustering` (Redis-adapter + présence Redis + sticky + tests cross-worker verts).
- **L12 `synchronous_commit`** : abandonné, **ne pas rejouer** (aucun gain mesuré).

**Livrable :** périmètre « ne pas toucher » explicite.

---

## 2. Récapitulatif du temps total

| Étape | Description | Temps |
| --- | --- | --- |
| 0 | Geler & archiver | ~10 min |
| 1 | Séparer reformatage et logique | ~15 min |
| 2 | Rejouer les 5 leviers proprement | ~2 h 05 |
| 3 | Réconcilier la doc | ~15 min |
| 4 | Post-mortem incident prod | ~25 min |
| 5 | Isoler le compose prod | ~20 min |
| 6 | Cadrer l'interdit / en attente | ~5 min |
| **Total** | **travail effectif assisté IA** | **~3 h 35** |

> **Soit ~une demi-journée** de travail concentré, hors temps d'exécution des tests de charge
> (qui tournent en parallèle). La majeure partie (étape 2) est du rejeu guidé, l'IA produisant
> les diffs minimaux et les messages de commit.

---

## 3. Nouveaux chantiers (B → G)

> Ces chantiers sont **indépendants de la refonte (A)** et constituent le reste à faire pour
> l'event. Le détail technique vit dans les docs dédiés ; ici = cadrage + priorité + reste à faire.

### Chantier B — Chaîne email → billet PDF  🔴 Haute
> Réf : [diagnostic email/billet/wallet](./diagnostics/email-billet-wallet.md) §2.2 / §3
> · [décision lib PDF/badge](./decisions/lib-pdf-badge.md) (choix moteur)

Aujourd'hui le PDF du billet est généré (Puppeteer→R2) **mais n'est pas joint à l'email**.

**Décision moteur PDF : 🟢 Option D — rendu déporté sur un service externe (Gotenberg sur Cloud Run).**
On sort **totalement** le rendu PDF du VPS (le goulot CPU mesuré) → charge jour-J neutralisée, fidélité
préservée (même moteur Chromium), **faisable sans attendre la migration GCP**.

| Action | Temps estimé |
| --- | --- |
| **Déployer Gotenberg sur Cloud Run** (image + config min-instances) | ~½ j |
| **Auth service-to-service** (IAM / clé) + egress VPS→GCP sécurisé | ~½ j |
| Brancher le back sur le service de rendu (remplacer l'appel Puppeteer local pour le billet) | ~½ – 1 j |
| Générer (ou réutiliser le cache R2) le PDF à la confirmation + brancher dans le flux | ~1 h 30 |
| Joindre le PDF à l'email + **lien de secours** (URL R2 signée) | ~1 h |
| Séquencer : email envoyé **seulement quand le billet est prêt** (billet → email) | ~1 h |
| Tests (idempotence, échec génération, fallback lien, cold-start) | ~1 h |
| **Sous-total B** | **~2 – 3 jours** |

**Effort : ~moyen-élevé** (nouveau service managé à exploiter, mais charge VPS éliminée).

> 💰 **Coût Cloud Run :** Gotenberg = 0 € de licence (open source). Compute pay-per-use ≈ **< 1 €**
> en scale-to-zero, **~5-12 €** si on garde 1-2 instances chaudes sur les 2 jours d'event (anti
> cold-start). Chiffrage détaillé : comparatif §6.
>
> ⚠️ **Garde-fou conservé :** le Puppeteer local **reste en place** comme fallback tant que le rendu
> Cloud Run n'est pas validé. Comparaison visuelle **pixel-à-pixel** avant de basculer en prod.
> Le **load test S4** doit inclure un scénario badge (le ~1,5 s/badge est encore une hypothèse).

### Chantier C — Migration ESP (délivrabilité)  🔴 Haute · ⏳ choix à trancher
> Réf : diagnostic §1 (ligne « Stack email »)

Email actuel = **nodemailer SMTP direct** (pas un ESP) → risque de délivrabilité sur ~12 400 envois.

| Action | Temps estimé |
| --- | --- |
| **Trancher l'ESP** (Brevo / Scaleway) | décision (non-dev) |
| Créer le compte + configurer le domaine d'envoi (SPF / DKIM / DMARC) | ~2 h *(+ propagation DNS, calendaire)* |
| Brancher l'ESP derrière nodemailer (transport) | ~1 h |
| Test d'envoi en masse représentatif + ajustements | ~1 h 30 |
| **Sous-total C (dev)** | **~4 h 30** |

> ⚠️ **Temps calendaire en plus** (pas du dev) : propagation DNS (~qq h → 24 h) et **warm-up IP/domaine**
> si gros volume (plusieurs jours d'envois progressifs). À anticiper **bien avant** l'ouverture des inscriptions.

**Effort dev : ~moyen** ; **le vrai enjeu = délivrabilité + calendrier de warm-up.** Plus gros risque client.

### Chantier D — Sécurité QR  🔸 Moyenne
> Réf : diagnostic §5

Le QR encode l'**UUID brut** de la registration, servi aussi sur une **route publique non
authentifiée** (`/qrcode/:id`, seule garde : longueur ≥ 10).

| Action | Temps estimé |
| --- | --- |
| **Signer le payload du QR (HMAC)** côté back (génération + secret) | ~1 h |
| Valider la **signature au scan** côté mobile **et** print-client (2 repos) | ~2 h |
| **Durcir / restreindre** la route publique `/qrcode/:id` | ~1 h |
| **Sous-total D** | **~4 h** |

**Effort : ~faible-moyen**, mais **touche 2 clients** (mobile + print-client) → coordination.
Sensible vu les **données nominatives MEAE**.

### Chantier E — Sauvegarde DB automatique  🔴 Haute · brief prêt
> Réf : [brief-backup-automatique-db.md](../../../backlog/a-faire/brief-backup-automatique-db.md)

Le script `db-dump.sh` existe mais est **manuel**. MVP (le 80/20) :

| Action | Temps estimé |
| --- | --- |
| **Cron** : 1 dump/jour en fenêtre creuse (wrapper autour du script existant) | ~1 h |
| **Offsite** : push de chaque dump hors VPS (R2) + chiffrement au repos | ~1 h 30 |
| **Restore testé** : 1 restauration prouvée sur staging + procédure documentée | ~1 h 30 |
| **Alerte** si un dump échoue | ~1 h |
| **Sous-total E (MVP)** | **~5 h** |
| *(Plus tard)* paliers de rétention grand-père/père/fils | ~2 h |

**Effort : ~demi-journée** pour le MVP.

### Chantier F — Continuité d'activité  � Partiellement reportée (post-GCP)
> Réf : [plan-continuite-activite.md](../../a-faire/infra-scaling-pca/plan-continuite-activite.md)

**Pourquoi la continuité est ultra importante :**
- C'est **la seule protection contre l'erreur humaine et la corruption logique** pendant l'event.
  La haute dispo (réplica) garde le service **debout**, mais elle **réplique fidèlement** une
  catastrophe : un `DELETE` accidentel ou une migration foireuse est copié sur le réplica en 1 s.
  Seul le **WAL/PITR** permet de **revenir à l'instant d'avant** (ex. T-15 min).
- L'enjeu est **contractuel et réputationnel** : pour un client **institutionnel MEAE** avec
  ~12 400 inscriptions, perdre ou corrompre les données **sans moyen de reprise** = catastrophe.

> **🔵 Décision : la HA (réplication streaming) est REPORTÉE après la migration GCP.**
> Sur **Cloud SQL** (Postgres managé GCP), la **haute dispo ET le PITR sont natifs** (réplica +
> backups automatiques + restauration point-in-time intégrés). Construire la réplication à la main
> sur le VPS maintenant serait **du travail jeté**. On l'obtient « gratuitement » avec GCP.

**Pour l'event (avant GCP), filet de sécurité minimal :**

| Action | Temps estimé | Statut |
| --- | --- | --- |
| **Backup MVP** (cf. chantier E) — dump auto + offsite + restore testé | cf. E | ✅ à faire |
| **Archivage WAL / PITR léger** sur le VPS (sans réplica HA) — « remonter le temps » | ~4 h | ✅ à faire si le temps le permet |
| ~~Réplication streaming + bascule HA~~ | ~2-3 j | 🔵 **reporté → migration GCP** |

**Conséquence :** F passe de « ~2-3 jours d'infra lourde » à **un sous-ensemble léger** (~½ jour de
PITR en plus du backup MVP). La HA complète arrive **avec GCP**, hors périmètre de ce mois.

> ⚠️ *À confirmer : timing de la migration GCP vs l'event (4-5 sept). Si GCP est livré avant l'event,*
> *la continuité est couverte nativement. Sinon, le couple backup MVP + PITR léger est le filet.*

### Chantier G — Wallet Apple + Google  ⚪ V2 (hors périmètre event)
> Réf : diagnostic §2.3 / §4

**Reporté en V2.** QR + PDF suffisent pour LFD 2026. À démarrer en V2 (avec délai de validation
Apple en tête). Prérequis comptes (Apple Developer, Google Wallet API) détenus par Rabiegha, **tout
à créer le moment venu**.

**Estimation indicative V2 : ~1 semaine de dev** (`.pkpass` signé Apple + Google Wallet API + intégration
file/R2 + tests), **hors délais comptes/certifs** (validation Apple = calendaire, plusieurs jours).

### Chantier I — ⚡ Levier débit n°1 : compteur capacité dénormalisé + transaction courte  🔴 Haute (perf)
> Réf : `attendee-ems-back/docs/infra/lfd-2026-load-test-results.md` §5 / §8 (recommandation prio 2)

**Pourquoi ça plafonne à ~30/s — c'est prouvé par nos propres mesures (25/06) :** ce n'est
**ni le CPU** (2→4 cœurs = même débit : 32,8 → 32,65/s), **ni le pool DB** (1–4 connexions
actives sur 100), **ni l'email** (worker séparé à ~8 % de CPU pendant la charge). C'est la
**transaction Prisma d'inscription** (`registerToEvent`, `public.service.ts`) qui tient une
connexion pendant un **`COUNT` de capacité** + upsert + create → **sérialisation** des
inscriptions concurrentes. C'est un plafond **algorithmique**, donc **améliorable sans changer
d'infra**.

**Le fix :** compteur de capacité **dénormalisé** (`UPDATE … SET pris = pris + 1 WHERE pris <
capacite` atomique, ou Redis `DECR`) + **transaction réduite au strict `create`** + garde
d'unicité (contrainte, `P2002`→409). Anti-surbooking préservé par la contrainte/verrou ciblé.
**Gain plausible : ×3 – ×10** sur le hot path (le débit suit alors le CPU/cluster au lieu du
verrou DB).

| Action | Temps estimé |
| --- | --- |
| Compteur dénormalisé (colonne/ligne de jauge + UPDATE atomique, ou Redis DECR + réconciliation) | ~½ j |
| Réduire la transaction au `create` + garde d'unicité ; refus propre « complet » (pas une 500) | ~½ j |
| Tests concurrence (zéro surbooking prouvé sous k6) + mesure avant/après | ~½ j |
| **Sous-total I** | **~1,5 j** |

> ⚠️ **C'est LE plus gros levier de débit du workstream.** Tous les autres leviers de perf déjà
> appliqués (pool 100, email async, transaction allégée, worker séparé, cluster) ne font que
> tourner autour de ce point de sérialisation : cluster 3× = +12 % seulement. Tant que I n'est
> pas fait, le plafond ~30-37/s reste, quelle que soit l'infra (OVH, GCP, VM plus grosse).
> À rejouer **après A** (base propre), sur la branche `feat/register-capacity-counter`.

### Chantier H — Inscriptions par session (refonte sessions)  🔴 Haute (fonctionnel)
> Réf : [sessions-inscriptions-lfd2026/README.md](../../a-faire/sessions-inscriptions-lfd2026/README.md)

**Besoin (cahier des charges) :** LFD comporte **plusieurs sessions** ; les inscriptions publiques
doivent pouvoir se faire **au niveau session** (lien public **dédié par session**), avec **capacité +
liste d'attente** et **contrôle organisateur** (stopper, bloquer, augmenter la capacité, promouvoir
depuis la waitlist). Flux : *inscription session → inscription event → check-in dans la session sur site*.

**Modèle acté :** la session a un **mode** (`registration_enabled`) — walk-in (check-ins only, pas de
lien) vs. avec inscription (lien public + capacité optionnelle + stats complètes). Token public
**dédié par session** (l'agrégateur « lien event unique » = phase 3). Anti-doublon **assoupli**
(1 registration event qui **accumule** les sessions).
🐛 **Bug à corriger** : `session_choice_ids` est accepté par le form public mais **jamais persisté**.

| Action | Temps estimé |
| --- | --- |
| **Phase 0** — migration Prisma (champs session + statut `SessionRegistration`) + backfill + fix bug `session_choice_ids` | ~½ j |
| **Phase 1** — inscription publique par session (token, routes publiques, anti-doublon assoupli, capacité + waitlist, contrôles manuels, stats) | ~2 – 3 j |
| **Phase 2** — refonte front sessions (cartes + page détail + config + liste inscrits + stats live, *mode-aware*) | ~3 – 5 j |
| **Sous-total H (phases 0–2)** | **~5,5 – 8,5 j** |
| 🔵 *Reporté post-event (phase 3) :* form/emails dédiés session, contenu riche, websocket live | ~2 – 3 j |

> ⚠️ **Impact capacité :** H est un **besoin fonctionnel** (pas du perf) → il **concurrence** les
> autres chantiers pour la capacité du mois. Phases 0–1 = **event-critique** (sans elles, pas
> d'inscription par session). Phase 2 (front) = **à arbitrer** selon le buffer ; a minima une **V1
> réduite** (cartes *mode-aware* + config lien/capacité + liste inscrits), le reste en phase 3.

---

## Récap des temps — périmètre du mois (fourchettes prudentes)

| Chantier | Temps dev estimé | Temps calendaire en plus |
| --- | --- | --- |
| **A** — Refonte propre §7 *(priorité n°1)* | **~1 – 2 jours** *(prudent)* | — |
| **I** — ⚡ Levier débit n°1 (compteur capacité + tx courte) | **~1,5 jours** | — |
| **0-CI** — CI/CD **minimal** | **~2 jours** | — |
| **0-MON** — Monitoring **minimal** | **~2 jours** | comptes outils (uptime/Sentry) |
| **B** — Email → billet PDF (**Gotenberg/Cloud Run**) | **~2 – 3 jours** | service Cloud Run (~< 1 – 12 €) |
| **C** — Migration ESP | **~2 – 3 jours** | propagation DNS + **warm-up (1-2 sem)** |
| **D** — Sécurité QR HMAC | **~1 – 2 jours** | — (coordination 2 repos clients) |
| **E** — Backup auto (MVP) + PITR léger | **~1,5 – 2 jours** | — |
| **H** — Inscriptions par session (phases 0–2) | **~5,5 – 8,5 jours** | — |
| **F** — HA réplication complète | 🔵 **reporté** | → migration GCP |
| **G** — Wallet (V2) | ~1 semaine *(hors event)* | validation Apple |
| **Total périmètre event (mois)** | **~18 – 24 jours-dev** | DNS + warm-up ESP + Cloud Run |

> **⚡ Lecture capacité :** ~18-24 jours-dev sur **2 devs × ~20 j ouvrés = ~40 j-dev de capacité**
> → **ça rentre encore** (≈ 45-60 % de la capacité), mais le **buffer se réduit** avec l'ajout du
> Chantier H (fonctionnel). Conditions : **tenir le descope** (HA → GCP, CD complet → post-event,
> wallet → V2) et traiter **H phase 2 (front) en V1 réduite**, variable d'ajustement du planning.
> Voir le **planning daté** ci-dessous.

---

## 4. Reste à faire — synthèse priorisée (tous chantiers)

> **Contexte :** échéance **1 mois**, **2 devs**. Périmètre **descopé** pour tenir le délai.

| Priorité | Chantier | Décision |
|---|---|---|
| 🔴 P0 | **A** — Refonte propre §7 | **Priorité n°1** — débloque le déploiement propre du reste |
| 🔴 P0 | **C** — ESP (Brevo/Scaleway) | ⚠️ **lancer le warm-up tout de suite** (délai calendaire) |
| 🔴 P0/P1 | **I** — ⚡ Levier débit n°1 (compteur capacité dénormalisé + tx courte) | **Le seul levier qui lève le plafond ~30/s** — juste après A, avant le load test S4 |
| 🔴 P1 | **0-MON** — Monitoring minimal (uptime + alerte + Sentry) | Gardé (réduit) |
| 🔴 P1 | **0-CI** — CI/CD **minimaliste** (build+test PR + deploy staging) | Gardé (réduit, **sans CD/rollback auto**) |
| 🔴 P1 | **E** — Backup auto MVP | Gardé (filet de sécurité event) |
| � P1 | **H** — Inscriptions par session | **Nouveau** besoin fonctionnel LFD — ph. 0–1 **event-critique** ; ph. 2 front **à arbitrer** (V1 réduite) |
| �🔸 P1 | **D** — Sécurité QR HMAC | **Gardé** (simple, données MEAE) |
| 🟠 P2 | **B** — Email → billet PDF | Gardé **si le client l'exige** pour l'event, sinon V2 |
| 🔵 — | **F** — HA réplication complète | **Reporté → après migration GCP** (HA/PITR natifs Cloud SQL) |
| ⚪ V2 | **G** — Wallet Apple + Google | Hors périmètre event |

> **🔵 Migration GCP** : absorbe la HA + le PITR managés. La continuité lourde n'est donc **pas** à
> construire à la main ce mois-ci — pour l'event, le filet = **backup MVP (+ PITR léger)**.

---

## Planning 1 mois — 2 devs (proposition datée)

> Hypothèse : 2 devs, ~4 semaines. Estimations **prudentes** (réalité incluse, pas best-case IA).
> ⚠️ **À lancer dès J1 (calendaire, pas du dev) :** création compte ESP + configuration DNS
> (SPF/DKIM/DMARC) + **warm-up** — sinon les ~12 400 emails risquent le spam.

| Semaine | **Dev 1 (back / infra)** | **Dev 2 (CI / clients / sécu)** |
|---|---|---|
| **S1** | **A — Refonte propre** (rejouer les leviers, redéployer proprement) · lancer setup ESP (DNS) | **0-CI minimal** (build+test sur PR + deploy staging) · **0-MON** uptime + alerte |
| **S2** | **I — Levier débit n°1** (compteur capacité + tx courte, mesure avant/après) · **E — Backup MVP** + **PITR léger** | **D — QR HMAC** (back + mobile + print-client) · finaliser CI minimal · monitoring back (Sentry + métriques CPU) |
| **S3** | **C — Intégration ESP** derrière nodemailer + test envoi en masse | **B — Email → billet PDF** : déployer **Gotenberg sur Cloud Run** + brancher le rendu + joindre PDF + lien secours |
| **S4** | **Load test de validation** sur prod réelle (**+ scénario badge / Cloud Run**) · checklist event · **buffer** | Tests bout-en-bout (scan QR signé, email+PDF, fallback Puppeteer) · runbook event · **buffer** |

> **➕ Intégration Chantier H (inscriptions par session) :** phase 0 (migration + fix bug) + phase 1
> (inscription publique par session, back) se logent en **S2–S3 côté Dev 1** (event-critique) ; elles
> réutilisent les leviers de charge de A (transaction courte, worker BullMQ). La **phase 2 (refonte
> front sessions)** s'étale **S3–S4 côté Dev 2** en **V1 réduite**, en puisant dans le buffer S4. Le
> confort (phase 3 : form/emails dédiés, contenu riche, websocket) est **reporté post-event**. Si le
> buffer ne suffit pas → **arbitrer** : livrer H phases 0–1 + une front minimale, reste en V2.

**Livré au client à l'échéance (1 mois) :**
- ✅ Base de code propre et redéployée (refonte A)
- ✅ CI/CD minimal (tests bloquants sur PR, déploiement staging)
- ✅ Monitoring + alerting (uptime, erreurs, CPU)
- ✅ Sauvegarde DB automatique testée + PITR léger
- ✅ QR signé (HMAC) sécurisé
- ✅ Email avec billet PDF (rendu **Gotenberg/Cloud Run**, hors VPS) + ESP en délivrabilité maîtrisée
- ✅ **Inscriptions par session** (lien public dédié, capacité + waitlist, contrôle organisateur) + refonte front sessions *(front phase 2 : V1 réduite selon buffer)*

**Explicitement reporté (à présenter comme V2 / post-GCP) :**
- 🔵 Haute dispo (réplication streaming) → **arrive avec la migration GCP**
- 🔵 CI/CD complet (déploiement auto + rollback) → après l'event
- 🔵 Sessions (phase 3) : formulaire/emails dédiés par session, contenu riche, websocket live → post-event
- ⚪ Wallet Apple / Google → V2

> **Tampon recommandé :** garder la **S4 majoritairement en buffer + validation**. Sur 1 mois à 2 devs,
> le risque n'est pas le volume de code mais les **surprises infra** (ESP, prod, tests de charge).

---

---

## 5. Ce qu'Attendee gagne (chantier A — refonte)

### 5.1 Sur la qualité du code et la maintenabilité
- **Diffs revisables** : chaque levier passe de « 517/−381 illisible » à quelques dizaines de
  lignes lisibles → revue de code réelle possible.
- **Revert granulaire** : si un levier régresse, on le retire seul sans casser les autres.
  Aujourd'hui, le commit fourre-tout `d3f3421` rend tout revert impossible sans dégât collatéral.
- **Historique git propre** : 1 commit = 1 intention. L'historique redevient une documentation
  fiable de « pourquoi ce changement existe ».

### 5.2 Sur la fiabilité et le risque
- **Risque de régression caché éliminé** : le bruit de reformatage masquait potentiellement des
  changements de logique. En isolant, on sait exactement ce qui change.
- **Prod protégée** : le compose prod (+52) est isolé et **non fusionné** tant que non revu →
  plus de risque de pousser en prod un changement non validé.
- **Incident capitalisé** : le 502 prod devient un post-mortem avec garde-fou → l'erreur de
  collision Docker ne peut plus se reproduire silencieusement.

### 5.3 Sur la capacité technique réelle (les leviers eux-mêmes)
- **Inscription plus rapide sous charge** : la transaction allégée (L9) sort le `COUNT` capacité
  et l'anti-doublon de la transaction → moins de contention sur le hot path d'inscription.
- **Hot path déchargé** : l'email/QR asynchrone (L7) + le worker séparable `PROCESS_ROLE` (L8)
  retirent le coût CPU des emails/badges du chemin de la requête → l'API encaisse mieux les pics.
- **Base mesurée et fiable** : on conserve les **vrais chiffres** (~25/s, plafond CPU) au lieu de
  chiffres optimistes erronés → décisions de scaling basées sur des faits.

### 5.4 Sur le pilotage projet
- **Traçabilité** : chaque levier rattaché à son workstream (`infra-scaling-pca`,
  `async-badge-email`, `api-scaling-clustering`) → on sait quoi est livré, quoi est en attente.
- **Périmètre clair** : ce qui est interdit (Voie B cluster) et abandonné (`synchronous_commit`)
  est explicite → pas de re-travail inutile ni de déploiement dangereux.
- **Confiance pour LFD 2026** : une base propre, mesurée et reproductible, prête à être rejouée
  en confiance pour le pic d'inscription de l'événement.

---

## 6. En une phrase (chantier A)

Appliquer le §7 transforme **une nuit de travail efficace mais brouillonne** en un **socle propre,
revisable et mesuré** : Attendee garde tous les gains de capacité (transaction allégée, async
email, worker séparable) **sans la dette** (diffs illisibles, revert impossible, risque prod),
pour environ **une demi-journée** de travail assisté IA.
