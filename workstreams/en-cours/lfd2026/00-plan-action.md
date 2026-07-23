# Plan d'action — LFD 2026 (refonte propre + chantiers à venir)

> **Objet :** plan d'action consolidé du workstream pour l'événement La Fabrique de la Diplomatie
> 2026 (MEAE, ~12 400 inscriptions, 4–5 sept 2026). Couvre **(P0) les fondations ultra-urgentes**
> (CI/CD, monitoring — inexistants aujourd'hui), (A) la **refonte propre** du §7 de l'audit, puis
> (B→H) les **nouveaux chantiers** (chaîne email/billet, sécurité QR, sauvegarde DB, continuité,
> wallet V2, **inscriptions par session**). Les durées supposent un travail **assisté par IA**.
> **Date :** 30 juin 2026 · **Rattachement :** workstream `lfd2026`
> **Références :** [audit-branche-staging.md](../gcp-migration/infra-scaling-pca/journal/2026-06-26-audit-branche-staging.md) ·
> [diagnostic email/billet/wallet](./B-email-billet-pdf/email-billet-wallet.md) ·
> [plan-continuite-activite.md](../gcp-migration/infra-scaling-pca/plan-continuite-activite.md) ·
> [brief-backup-automatique-db.md](../../../backlog/fait/brief-backup-automatique-db.md)

---

## Vue d'ensemble des chantiers

> **Cadre :** échéance **1 mois**, **2 devs**. Périmètre **descopé** pour tenir (cf. §4 + planning).
> **📊 Avancement vivant (%, temps, jalons) → [03-suivi-chantiers.md](./03-suivi-chantiers.md)** ·
> rapports hebdo manager → [rapports/](./rapports/2026-W28-rapport-hebdo.md).

| #         | Chantier                                                                                                                                | Priorité                       | Statut                                                                           | Détail                                                                                |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **A**     | **Refonte propre (§7 audit)** — rejouer les leviers proprement                                                                          | 🔴 **Priorité n°1**            | À faire                                                                          | §1–§2 ci-dessous                                                                      |
| **I**     | **⚡ Levier débit n°1 — compteur capacité dénormalisé + transaction courte** (lève le plafond ~30/s)                                    | 🔴 **Haute (perf)**            | 🟡 Diagnostiqué, non codé                                                        | §3-I                                                                                  |
| **C1–C3** | Migration Mailgun + délivrabilité (~12 400 envois)                                                                                     | 🔴 Haute                       | 🟡 C1/C2 faits, C3 en cours                                                       | §3-C                                                                                  |
| **C4**    | **Fallback SMTP résilient** — transactionnel critique, anti-doublon, quota minute/jour                                                  | 🟠 P1 résilience               | ⚪ Cadré, mode manuel d'abord                                                     | [C4](./C-migration-esp/c4-fallback-smtp-resilient.md)                                 |
| **0-MON** | Monitoring & alerting **minimal** (uptime / Sentry / CPU)                                                                               | 🔴 Haute                       | ✅ Prod opérationnel                                                            | §P0                                                                                   |
| **0-CI**  | CI/CD **minimaliste** (build+test PR + deploy staging)                                                                                  | 🔴 Haute                       | 🟢 Chaîne validée, nettoyage/gouvernance restants                                | §P0                                                                                   |
| **E**     | Sauvegarde DB automatique (cron + offsite + restore testé)                                                                              | 🔴 Haute                       | ✅ **Terminé** — [PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15) | §3-E                                                                                  |
| **D**     | Sécurité QR (signature HMAC + durcir route publique)                                                                                    | 🔸 Moyenne                     | **Gardé** (simple)                                                               | §3-D                                                                                  |
| **B0**    | **Email → billet PDF (client Gotenberg)** — auth OIDC vers Cloud Run privé                                                              | 🔴 Haute                       | 🟢 Client mergé `staging`, smoke runtime restant                                 | §3-B                                                                                  |
| **B1**    | **Email → billet PDF (branchement)** — retries Gotenberg + fallback Puppeteer compatible                                               | 🔴 Haute                       | 🟢 Branche prête, CI/merge/smoke restants                                        | §3-B                                                                                  |
| **B2**    | **Fallback PDF Puppeteer résilient** — worker async, circuit breaker, concurrence/budget bornés                                         | 🟠 P1 résilience               | ⚪ Cadré, après B1 et avec C2.1                                                   | [B2](./B-email-billet-pdf/B2-fallback-puppeteer-resilient.md)                         |
| **H**     | **Inscriptions par session** (lien public par session + capacité/waitlist + refonte front)                                              | 🔴 Haute (**fonctionnel**)     | 🟡 Cadré, prêt à découper                                                        | §3-H                                                                                  |
| **J**     | **Capacité live forte charge + J-ENTREES** — réservations et entrées physiques live + **pic combiné** insc/check-in                       | 🔴 Haute (**event/CDC**)       | 🟢 Implémentation livrée (PR back #67 + PR front #26) ; exécution k6 check-in/combiné reportée                   | [J](./J-capacite-live/README.md)                                                       |
| **BIL**   | **Plateforme billetterie LFD** — landing pages + **backoffice client RBAC et dashboard entrées**                                        | 🔴 Haute (**produit**)         | 🟡 **~35 %, 7–11 j-dev** — Corentin ; dépend de H+J+C2.1/B                      | §3-BIL                                                                                |
| **K**     | **Résilience event** — checklist finale : être sûr que **tout ce qui protège** est en place (saturation, disque, perte, recovery testé) | 🔴 Haute (**event**)           | 🔴 À vérifier avant J-7                                                          | [ws résilience](../../a-faire/resilience-event-lfd2026/README.md)                     |
| **M**     | **Scan mobile et recette terrain** — offline, multi-scanner, feedback et répétition                                                     | 🔴 Haute (**terrain**)         | 🔴 **~10 %, 5,5–9 j-dev** ; GO terrain non acquis                              | [M](./M-appli-mobile-lfd2026/README.md)                                                |
| **N**     | **Architecture event-ready LFD** — coordination sync/async, anti-bot, charge et runbooks                                                | 🔴 Haute (**event**)           | 🟡 Coordinateur ; ne pas doubler les lots contributeurs                         | [N](./N-architecture-event-ready/README.md)                                           |
| **O**     | **Audit métier et sécurité LFD** — accès/actions BIL, avant/après, E2E et preuve CDC                                                     | 🔴 Haute (**CDC**)             | 🟡 **~10 %, 5–8,5 j bruts** ; hooks partagés avec BIL/T                         | [O](./O-audit-lfd/README.md)                                                          |
| **F**     | API stateless, multi-instance et HA                                                                                                     | 🔵 **Post-event · 15–27 j-dev** | audit puis séparation des rôles, état partagé, validation multi-instance et GCP | §3-F                                                                                  |
| **L**     | Architecture event-driven globale                                                                                                       | 🔵 **Post-event · 27–46 j-dev** | outbox/inbox, contrats et migration incrémentale                                | [L](./L-architecture-event-driven/README.md)                                          |
| **P**     | Plateforme de journalisation Attendee                                                                                                    | 🔵 **Post-event**              | **15–25 j après L0/L1** ; audit et logs techniques séparés                      | [P](./P-plateforme-journalisation/README.md)                                          |
| **G**     | Wallet Apple + Google — **harnais dans B · onboarding now · Google fast-follow · Apple hors chemin critique**                           | 🟡 Découpé (was ⚪ V2)         | Onboarding à lancer J1                                                           | §3-G                                                                                  |

> **⚡ Ordre :** **A (refonte) d'abord** — elle débloque le déploiement propre de tout le reste.
> En parallèle, lancer **dès J1** le warm-up ESP (délai calendaire). **CI/CD et monitoring restent
> volontairement minimalistes** pour tenir le mois. La **HA complète est reportée à la migration GCP**
> (Cloud SQL fournit HA + PITR managés → inutile de la construire à la main maintenant). G = V2.

---

## ⚡ P0 — Fondations minimales (CI/CD + monitoring, version réduite)

> Ces deux chantiers sont **transverses** : ils protègent **tous** les autres. En contexte 1 mois,
> on les garde en **version minimale** — assez pour ne pas voler à l'aveugle sur la prod, sans
> y engloutir le planning, à quelques semaines d'un événement institutionnel MEAE à fort enjeu.

### Chantier 0-CI — CI/CD (minimal) 🔴 Haute

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

| Action                                                                                     | Temps estimé |
| ------------------------------------------------------------------------------------------ | ------------ |
| **CI** : sur chaque PR → lint + build + tests (back & front), **bloque le merge si rouge** | ~1 jour      |
| **Déploiement staging** semi-auto + **healthcheck post-deploy**                            | ~1 jour      |
| Gestion des **secrets** (fin des `.env` copiés à la main)                                  | ~2 h         |
| **Sous-total 0-CI (minimal)**                                                              | **~2 jours** |
| 🔵 _Reporté post-event :_ CD prod complet (déploiement auto + rollback)                    | ~1-2 j       |

> **Décision :** on garde le **déploiement prod manuel** ce mois-ci, mais encadré par une
> **checklist pré-déploiement** (pour éviter un nouveau 502 type collision Docker). Le CD complet
> (auto + rollback) est **reporté après l'event**.

### Chantier 0-MON — Monitoring & alerting (minimal) 🔴 Haute

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

| Action                                                                                                                                                        | Temps estimé  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| **Uptime externe + alerte** (UptimeRobot / Better Uptime) sur `/health` → SMS/Slack si down                                                                   | ~2 h          |
| **Error-tracking** (Sentry) back + front → exceptions remontées                                                                                               | ~demi-journée |
| **Métriques système** (CPU/RAM/disque) + dashboard (Grafana+node-exporter, ou Netdata)                                                                        | ~1 jour       |
| **Alerting sur seuils** — **CPU élevé · file BullMQ qui gonfle · 🔴 disque ≥ 70/80/90 %** (disque plein = Postgres ne peut plus écrire le WAL → **API down**) | ~demi-journée |
| **Sous-total 0-MON**                                                                                                                                          | **~2 jours**  |

> 🔴 **L'alerte disque est la protection n°1 souvent oubliée.** Un disque plein est presque **toujours
> un échec de monitoring** (on n'a pas été prévenu). La HA **n'aide pas** (le réplica se remplit pareil).
> À coupler : **rotation des logs** (`logrotate` / Docker `max-size`), **offload PDF+backups vers R2**
> (pas sur le disque local), **surveiller les slots de réplication bloqués** (WAL jamais recyclé). Cf.
> [décision — tenir sans GCP §4.3](./K-resilience-event/tenir-event-sans-gcp.md) et le workstream **résilience event**.

> **⚡ Ces deux chantiers (~4 jours, version minimale) sont des fondations à poser tôt** (S1), en
> parallèle de la refonte (A). Tant qu'ils ne sont pas là, chaque modif sur la prod se fait
> **sans filet de sécurité**.

---

## 0. Principe directeur (chantier A — refonte)

> **📎 Fichiers liés (chantier A) :** [01-suivi-leviers.md](./A-I-leviers/01-suivi-leviers.md) (état vivant par levier) ·
> [audit branche staging](../gcp-migration/infra-scaling-pca/journal/2026-06-26-audit-branche-staging.md) ·
> [NOW.md](../../../workspace-rabie/NOW.md) · [index des learnings](../../../learnings/README.md).

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

| Ordre | Levier                                                                                   | Branche                    | Nature                              | Temps estimé |
| ----- | ---------------------------------------------------------------------------------------- | -------------------------- | ----------------------------------- | ------------ |
| 1     | **L9b** transaction session allégée (`registerToSession`) — diff minimal + test de non-régression | `feat/register-session-tx-slim` | rejeu à la main | **~½–1 j** |
| 1b    | **L9a** transaction event allégée (`registerToEvent`) — ne pas abandonner le L9 classique | `feat/register-event-tx-slim` | rejeu à la main | **~½ j** |
| 2     | **L7** email async BullMQ                                                                | `feat/email-async-bullmq`  | rejeu guidé (nouveaux fichiers)     | **~30 min**  |
| 3     | **L8** worker `PROCESS_ROLE` (Voie A)                                                    | `feat/process-role-worker` | rejeu à la main (gating `main.ts`)  | **~20 min**  |
| 4     | **L3** `directUrl` Prisma                                                                | `chore/prisma-directurl`   | cherry-pick ciblé (+5 lignes)       | **~10 min**  |
| 5     | **L1/L2/L10** stack staging + pgBouncer + pool                                           | `infra/staging-stack`      | cherry-pick (déjà propre)           | **~20 min**  |

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

| Étape     | Description                      | Temps       |
| --------- | -------------------------------- | ----------- |
| 0         | Geler & archiver                 | ~10 min     |
| 1         | Séparer reformatage et logique   | ~15 min     |
| 2         | Rejouer les 5 leviers proprement | ~2 h 05     |
| 3         | Réconcilier la doc               | ~15 min     |
| 4         | Post-mortem incident prod        | ~25 min     |
| 5         | Isoler le compose prod           | ~20 min     |
| 6         | Cadrer l'interdit / en attente   | ~5 min      |
| **Total** | **travail effectif assisté IA**  | **~3 h 35** |

> **Soit ~une demi-journée** de travail concentré, hors temps d'exécution des tests de charge
> (qui tournent en parallèle). La majeure partie (étape 2) est du rejeu guidé, l'IA produisant
> les diffs minimaux et les messages de commit.

---

## 3. Nouveaux chantiers (B → G)

> Ces chantiers sont **indépendants de la refonte (A)** et constituent le reste à faire pour
> l'event. Le détail technique vit dans les docs dédiés ; ici = cadrage + priorité + reste à faire.

### Chantier B — Chaîne email → billet PDF 🔴 Haute

> **📎 Fichiers liés :** [diagnostic email/billet/wallet §2.2 / §3](./B-email-billet-pdf/email-billet-wallet.md) ·
> [décision lib PDF/badge](./B-email-billet-pdf/lib-pdf-badge.md) ·
> [B2 fallback Puppeteer résilient](./B-email-billet-pdf/B2-fallback-puppeteer-resilient.md)

Aujourd'hui le PDF du billet est généré (Puppeteer→R2) **mais n'est pas joint à l'email**.

**Décision moteur PDF : Gotenberg privé sur Cloud Run en primaire, Puppeteer local conservé en
secours contrôlé.** Le rendu nominal sort du VPS, mais le fallback ne doit pas pouvoir saturer le
processus API pendant une panne Gotenberg.

> 🧩 **Harnais commun wallet (décision 2026-07-08) :** concevoir le service de génération Cloud Run avec
> une **interface agnostique** `generate(format, data)` — **PDF = 1ʳᵉ implémentation**, wallet Apple/Google
> = formats **pluggables** plus tard (cf. **G-harnais**). Le coût du harnais est payé **une fois** ici → pas
> de refacto quand on ajoutera les wallets. Le check-in lit le **même token QR** quel que soit le format.

> ✂️ **Découpage actuel :** B0 = client OIDC isolé ; B1 = branchement compatible + retries ;
> B2 = exécution asynchrone et protection du fallback. L'orchestration PDF→R2→email reste partagée
> avec C2.1 pour ne pas créer deux files PDF.

#### B0 — Client Gotenberg privé · **~½–1 j**

| Action | Statut |
| --- | --- |
| Cloud Run Gotenberg privé, invoker OIDC | ✅ Déployé/validé côté GCP |
| Client Nest OIDC, timeout, validation `%PDF`, erreurs typées | ✅ Mergé `staging` — B0 `f37df97` |
| Credential/config runtime et smoke réel depuis OVH | ⏳ Reste |

#### B1 — Branchement compatible · **~1,5–2 j**

| Action | Statut |
| --- | --- |
| Brancher les 3 points de rendu sur Gotenberg | ✅ Branche `feat/pdf-generation-hardening` |
| Retry transitoire borné + fallback Puppeteer | ✅ Code/tests `9ed7321` |
| Fidélité JS/polices/dimensions | ✅ Couvert dans B1, smoke réel restant |
| CI, PR/merge explicite et smoke staging | ⏳ Reste |

#### B2 — Fallback Puppeteer résilient · **~2–4 j bruts, ~1–2 j nets après C2.1**

- queue `pdf.generate` et worker séparé de l'API ;
- concurrence Puppeteer basse, timeout, fermeture `finally` et récupération browser ;
- circuit breaker Gotenberg, cooldown/canari et budgets de fallback ;
- publication R2 atomique puis `email.send` ;
- idempotence par `registration_id`, sans `ticket_id` ni `ticket_version` métier.

**Effort : moyen-élevé**, avec recouvrement explicite entre B2 et le worker PDF de C2.1.

> 💰 **Coût Cloud Run :** Gotenberg = 0 € de licence (open source). Compute pay-per-use ≈ **< 1 €**
> en scale-to-zero, **~5-12 €** si on garde 1-2 instances chaudes sur les 2 jours d'event (anti
> cold-start). Chiffrage détaillé : comparatif §6.
>
> ⚠️ **Gate :** test de charge Gotenberg forcé KO. La latence d'inscription doit rester stable et
> la lane Puppeteer ne doit pas dépasser son budget CPU/RAM.

### Chantier C — Mailgun, délivrabilité et fallback SMTP 🔴 Haute

> **📎 Fichiers liés :** [diagnostic email/billet/wallet §1](./B-email-billet-pdf/email-billet-wallet.md) ·
> [décision ESP — Mailgun](./C-migration-esp/esp-choix-mailgun.md) ·
> [C3 warm-up](./C-migration-esp/c3-warmup-delivrabilite.md) ·
> [C4 fallback SMTP résilient](./C-migration-esp/c4-fallback-smtp-resilient.md)

Mailgun API EU est maintenant le transport primaire en staging/prod. Le SMTP OVH historique reste
configuré en production, mais il n'existe aujourd'hui aucun failover automatique.

| Lot | État / reste |
| --- | --- |
| **C1** — compte Mailgun EU + SPF/DKIM/DMARC/webhooks | ✅ Terminé |
| **C2** — transport API, BullMQ, retries, throttle, idempotence, événements | ✅ Terminé sur `staging` |
| **C2.1** — PDF puis email après inscription | 🟡 MVP email livré, PDF/page de suivi restants |
| **C3** — warm-up et délivrabilité réelle | 🟡 En cours, calendaire |
| **C4** — SMTP de secours borné | ⚪ ~2–3 j : handoff durable, anti-doublon, quota et circuit breaker |

Le C4 ne remplace pas C3 et n'envoie jamais les lots warm-up/newsletter par SMTP. Il réserve le
relais OVH aux messages transactionnels critiques, commence en mode manuel et ne passe en auto
qu'après validation du quota, des timeouts ambigus et de la délivrabilité SMTP réelle.

**Effort dev C4 : ~2–3 j** ; le vrai enjeu reste la prévention des doublons et la conservation d'un
budget SMTP utile pendant un incident long.

### Chantier D — Sécurité QR 🔸 Moyenne

> **📎 Fichiers liés :** [diagnostic email/billet/wallet §5](./B-email-billet-pdf/email-billet-wallet.md)

Le QR encode l'**UUID brut** de la registration, servi aussi sur une **route publique non
authentifiée** (`/qrcode/:id`, seule garde : longueur ≥ 10).

| Action                                                                     | Temps estimé |
| -------------------------------------------------------------------------- | ------------ |
| **Signer le payload du QR (HMAC)** côté back (génération + secret)         | ~1 h         |
| Valider la **signature au scan** côté mobile **et** print-client (2 repos) | ~2 h         |
| **Durcir / restreindre** la route publique `/qrcode/:id`                   | ~1 h         |
| **Sous-total D**                                                           | **~4 h**     |

**Effort : ~faible-moyen**, mais **touche 2 clients** (mobile + print-client) → coordination.
Sensible vu les **données nominatives MEAE**.

### Chantier E — Sauvegarde DB automatique ✅ TERMINÉ · [PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15)

> **📎 Fichiers liés :** [brief backup automatique DB](../../../backlog/fait/brief-backup-automatique-db.md) ·
> [scripts/backup/](https://github.com/Rabiegha/attendee-ems-back/tree/feature/db-backup/scripts/backup)

**Livraison complète — dépasse le brief initial sur 3 points.**

| Action                                                                                                       | Temps estimé | Statut                                                |
| ------------------------------------------------------------------------------------------------------------ | ------------ | ----------------------------------------------------- |
| **Cron** : timer systemd `ems-db-backup` 03:00, `Persistent=true`                                            | ~1 h         | ✅ Livré                                              |
| **Offsite** : gpg AES256 + rclone R2, checksum vérifié post-upload                                           | ~1 h 30      | ✅ Livré                                              |
| **Restore testé** : test automatisé quotidien 04:00 + exercice sinistre DR (`DROP SCHEMA CASCADE`) + runbook | ~1 h 30      | ✅ **Dépassé** (7 nuits de run, RTO=287–303 s mesuré) |
| **Alerte** si dump/restore échoue                                                                            | ~1 h         | ✅ Livré (email SMTP, 2 destinataires)                |
| **Sous-total E (MVP)**                                                                                       | **~5 h**     | ✅                                                    |
| ~~_(Plus tard)_~~ paliers rétention GFS locaux (7j/4sem/3mois) + tiering R2 (30j/12sem/~13mois)              | ~2 h         | ✅ **Anticipé et livré**                              |

> **Résultats après 7 jours de run automatique :** 7/7 backups OK · 7/7 restore-tests VERTS ·
> RTO 287–303 s · 0 échec réel · disque VPS plafonné à ~40 Go (tiering R2 porte le long terme).

> **Non livré dans E (appartient à F) :** archivage WAL / PITR léger — non fait, marqué
> « si le temps le permet » dans §3-F. Filet de sécurité assuré par le backup MVP.

**Effort réel : ~2 jours** (MVP + paliers + tests + bugs fixes alerting).

### Chantier F — API stateless, multi-instance et continuité/HA 🔵 Post-event

> **📎 Fichiers liés :** [plan de continuité d'activité](../gcp-migration/infra-scaling-pca/plan-continuite-activite.md) ·
> [PCA LFD 2026](../../../infra/lfd-2026-pca.md) · [stratégie SLA](../../../infra/lfd-2026-sla-strategy.md) ·
> [workstream migration GCP (archi cible B)](../../a-faire/gcp-migration/README.md)
> · [cadrage stateless et multi-instance](./F-stateless-multi-instance/README.md)

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

| Action                                                                              | Temps estimé | Statut                                                                           |
| ----------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------- |
| **Backup MVP** (cf. chantier E) — dump auto + offsite + restore testé               | cf. E        | ✅ **Terminé** ([PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15)) |
| **Archivage WAL / PITR léger** sur le VPS (sans réplica HA) — « remonter le temps » | ~4 h         | ✅ à faire si le temps le permet                                                 |
| ~~Réplication streaming + bascule HA~~                                              | ~2-3 j       | 🔵 **reporté → migration GCP**                                                   |

**Conséquence :** avant LFD, F se limite au filet de continuité et à la cartographie. Après LFD,
F regroupe le travail applicatif stateless (sessions, WebSockets, workers, impression, fichiers,
schedulers, pools) nécessaire au multi-instance, puis la HA complète avec GCP.

> ⚠️ _À confirmer : timing de la migration GCP vs l'event (4-5 sept). Si GCP est livré avant l'event,_
> _la continuité est couverte nativement. Sinon, le couple backup MVP + PITR léger est le filet._

> 🎯 **Décision (2026-07-08) : PAS de HA pour l'event — risque assumé.** Si la machine meurt jour-J
> → **down + recovery manuel** depuis backup (RTO borné, minutes → 1-2 h). C'est un **choix** : la HA
> couvre le risque **le moins probable** (panne matérielle sur 2 j) et **ne couvre PAS** les vrais dangers
> — **saturation** (→ L9+file), **disque plein** (→ 0-MON alerte disque), **perte/corruption** (→ backup).
> **⚠️ À vérifier avant de valider :** **pas de SLA d'uptime contractuel** imposé par le MEAE. Si SLA strict
> → accélérer GCP **ou** monter un failover OVH temporaire pour les 2 jours. Cf.
> [décision — tenir sans GCP §4.3](./K-resilience-event/tenir-event-sans-gcp.md).

### Chantier G — Wallet Apple + Google 🟡 Découpé (harnais now · onboarding now · Apple hors chemin critique)

> **📎 Fichiers liés :** [diagnostic email/billet/wallet §2.3 / §4](./B-email-billet-pdf/email-billet-wallet.md)

**Décision 2026-07-08 :** « faire Apple + Google en même temps que le PDF » = **vrai pour le harnais, faux
pour la génération par format**. On **découpe G en 4** au lieu d'un bloc V2 monolithique :

| Sous-chantier                                                                             | Nature                    | Quand                                      | Effort                     |
| ----------------------------------------------------------------------------------------- | ------------------------- | ------------------------------------------ | -------------------------- |
| **G-harnais** — interface `generate(format, data)` agnostique dans le service Cloud Run   | dev (fusionné dans **B**) | **maintenant** (avec B)                    | inclus dans B (~0 en plus) |
| **G-onboarding** — compte **Apple Developer** + **Pass Type ID cert** + **Issuer Google** | admin / délai externe     | **démarrer aujourd'hui**                   | 0 dev (pur calendaire)     |
| **G-Google** — objet `EventTicketObject` + **JWT signé** + lien « Save to Google Wallet » | dev (léger)               | **fast-follow après le PDF** si buffer     | ~½ – 1 j                   |
| **G-Apple** — `.pkpass` **signé** (manifest + PKCS#7) + assets @2x/@3x + gestion cert     | dev + op                  | **après PDF solide, hors chemin critique** | ~2 – 3 j                   |
| **Sous-total G (Google + Apple)**                                                         |                           |                                            | **~2,5 – 4 j**             |

**Pourquoi cette découpe :**

- **PDF** = aucune dépendance, couvre **100 %** des gens → must-have. Le wallet est un **confort**.
- **Google** = léger une fois l'**Issuer** approuvé (JSON + JWT) → bon candidat fast-follow.
- **Apple** = **compte dev (99 $/an) + Pass Type ID cert + renouvellement annuel** → **délai externe** +
  nouveaux modes de panne (secrets, expiration cert) → **jamais sur le chemin critique** de l'event.
- Le **check-in est découplé du format** (même token QR) → le harnais commun est le vrai gain.

> ⚠️ **Ordre :** G-harnais **dans B** (payé une fois) · G-onboarding **lancé maintenant** (délai) ·
> G-Google puis G-Apple **empilés après** que le PDF soit prouvé sous charge. Le PDF reste le **fallback 100 %**.

### Chantier I — ⚡ Levier débit n°1 : compteur capacité dénormalisé + transaction courte 🔴 Haute (perf)

> **📎 Fichiers liés :** `attendee-ems-back/docs/infra/lfd-2026-load-test-results.md` §5/§8 ·
> [décision — tenir l'event sans GCP (+ gate post-L9)](./K-resilience-event/tenir-event-sans-gcp.md) ·
> [carte stratégique 2 chantiers](../../../infra/lfd-2026-strategie-2-chantiers.md) ·
> [learning — pic inscription temps réel](../../../learnings/2026-07-08-pic-inscription-temps-reel.md) ·
> [plan de test de charge](../../../infra/lfd-2026-load-test-plan.md)

**Pourquoi ça plafonne à ~30/s — c'est prouvé par nos propres mesures (25/06) :** ce n'est
**ni le CPU** (2→4 cœurs = même débit : 32,8 → 32,65/s), **ni le pool DB** (1–4 connexions
actives sur 100), **ni l'email** (worker séparé à ~8 % de CPU pendant la charge). C'est la
**transaction Prisma d'inscription** (`public.service.ts`) qui tient une connexion pendant un
controle de capacité + upsert + create → **sérialisation** des inscriptions concurrentes. Elle existe
sur deux chemins à traiter séparément : **L9a** `registerToEvent` (event classique) et **L9b**
`registerToSession` (session LFD réelle, utilisée par `back-office-event`, avec verrou/COUNT session).
C'est un plafond **algorithmique**, donc **améliorable sans changer d'infra**.

**Le fix :** compteur de capacité **dénormalisé** (`UPDATE … SET pris = pris + 1 WHERE pris <
capacite` atomique, ou Redis `DECR`) + **transaction réduite au strict nécessaire** + garde
d'unicité (contrainte, `P2002`→409). Anti-surbooking préservé par la contrainte/verrou ciblé.
**Gain plausible : ×3 – ×10** sur le hot path (le débit suit alors le CPU/cluster au lieu du
verrou DB).

| Action                                                                                          | Temps estimé |
| ----------------------------------------------------------------------------------------------- | ------------ |
| **L9b session** : compteur/portier capacité session + transaction `registerToSession` raccourcie | ~½–1 j       |
| **L9a event** : compteur capacité event + transaction `registerToEvent` raccourcie              | ~½ j         |
| Tests concurrence (zéro surbooking prouvé sous k6) + mesure avant/après                         | ~½ j         |
| **Sous-total I**                                                                                | **~1,5–2 j** |

> ⚠️ **C'est LE plus gros levier de débit du workstream.** Tous les autres leviers de perf déjà
> appliqués (pool 100, email async, transaction allégée, worker séparé, cluster) ne font que
> tourner autour de ce point de sérialisation : cluster 3× = +12 % seulement. Tant que I n'est
> pas fait, le plafond ~30-37/s reste, quelle que soit l'infra (OVH, GCP, VM plus grosse).
> À rejouer **après A** (base propre), sur la branche `feat/register-capacity-counter`.

### Chantier H — Inscriptions par session (refonte sessions) 🔴 Haute (fonctionnel)

> **📎 Fichiers liés :** [workstream inscriptions/sessions — spec complète](../../a-faire/sessions-inscriptions-lfd2026/README.md) ·
> [02 — capacité live forte charge](../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md) ·
> [rabbit-hole — billetterie publique app vs module](../../../rabbit-holes/en-cours/billeterie-publique-app-vs-module.md)

**Besoin (cahier des charges) :** LFD comporte **plusieurs sessions** ; les inscriptions publiques
doivent pouvoir se faire **au niveau session** (lien public **dédié par session**), avec **capacité +
liste d'attente** et **contrôle organisateur** (stopper, bloquer, augmenter la capacité, promouvoir
depuis la waitlist). Flux : _inscription session → inscription event → check-in dans la session sur site_.

**Modèle acté :** la session a un **mode** (`registration_enabled`) — walk-in (check-ins only, pas de
lien) vs. avec inscription (lien public + capacité optionnelle + stats complètes). Token public
**dédié par session** (l'agrégateur « lien event unique » = phase 3). Anti-doublon **assoupli**
(1 registration event qui **accumule** les sessions).
🐛 **Bug à corriger** : `session_choice_ids` est accepté par le form public mais **jamais persisté**.

| Action                                                                                                                                         | Temps estimé     |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| **Phase 0** — migration Prisma (champs session + statut `SessionRegistration`) + backfill + fix bug `session_choice_ids`                       | ~½ j             |
| **Phase 1** — inscription publique par session (token, routes publiques, anti-doublon assoupli, capacité + waitlist, contrôles manuels, stats) | ~2 – 3 j         |
| **Phase 2** — refonte front sessions (cartes + page détail + config + liste inscrits + stats live, _mode-aware_)                               | ~3 – 5 j         |
| **Ajout CDC H-D8** — maximum deux activités `confirmed`/jour Europe/Paris + E2E concurrence                                                   | **~1 – 2 j**     |
| **Sous-total H (phases 0–2 + H-D8)**                                                                                                           | **~6,5 – 10,5 j** |
| 🔵 _Reporté post-event (phase 3) :_ form/emails dédiés session, contenu riche, websocket live                                                  | ~2 – 3 j         |

> ⚠️ **Impact capacité :** H est un **besoin fonctionnel** (pas du perf) → il **concurrence** les
> autres chantiers pour la capacité du mois. Phases 0–1 = **event-critique** (sans elles, pas
> d'inscription par session). Phase 2 (front) = **à arbitrer** selon le buffer ; a minima une **V1
> réduite** (cartes _mode-aware_ + config lien/capacité + liste inscrits), le reste en phase 3.

### Chantier J — Capacité live forte charge (portier Redis + WebSocket) 🔴 Haute (event)

> **📎 Fichiers liés :** [02 — capacité live forte charge (spec)](../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md) ·
> [learning — pic inscription temps réel](../../../learnings/2026-07-08-pic-inscription-temps-reel.md) ·
> [learning — scaling horizontal stateless](../../../learnings/2026-07-08-scaling-horizontal-stateless.md) ·
> [carte stratégique 2 chantiers](../../../infra/lfd-2026-strategie-2-chantiers.md) ·
> [plan de test de charge](../../../infra/lfd-2026-load-test-plan.md)

**Besoin :** tenir **3000 inscriptions quasi simultanées** sur sessions à capacité serrée, avec **statut
live** (pleine/fermée) et [dashboard des entrées physiques](./J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd),
**sans survente** ni saturation Postgres. Sépare **lecture** (statut → cache/WebSocket ou polling
authentifié) et **écriture métier synchrone** (inscription/scan → transaction PostgreSQL courte et atomique).
Après le commit, BullMQ ne traite que les effets secondaires. La réservation d'une place et le check-in
ne passent jamais dans un worker. Le chantier couvre le **pic combiné** inscriptions + check-ins et inclut
le levier candidat **L9.1** (compteur présence session O(1)).

| Action                                                                      | Temps estimé     |
| --------------------------------------------------------------------------- | ---------------- |
| Portier Redis atomique (anti-survente) + init/réconciliation                | ~1 j             |
| Gateway WebSocket + rooms + émission au changement de statut (back + front) | ~1,5 – 2 j       |
| Endpoint read servi par Redis + pré-chauffe + single-flight                 | ~1 j             |
| Isolation check-in (pool DB réservé) + L9.1 compteur O(1)                   | ~1 j             |
| J-ENTREES : snapshot event, live après scan et export présent/absent        | ~1 – 2 j net     |
| Test de charge **combiné** (insc + check-ins) + critères de gate            | ~1 – 1,5 j       |
| **Sous-total J réévalué**                                                   | **~5 – 9 jours** |

> ⚠️ **À démarrer APRÈS chantier A** (les leviers) et **après mesure** (les leviers + Gotenberg peuvent
> suffire). Recoupe partiellement **H** (capacité/waitlist) et **I** (compteur capacité) — ne pas double-compter.

### Chantier BIL — Plateforme billetterie (landing pages + backoffice client) 🔴 Haute (produit)

> **📎 Suivi vivant :** [03-suivi-chantiers.md](./03-suivi-chantiers.md) · **Owner :** Corentin · **Démarré :** 2026-07-07.

**Besoin :** offrir au client une **plateforme billetterie LFD** utilisable pour l'événement :

- **Gestion de la billetterie LFD** (inscriptions, billets, parcours session).
- **Page/landing publique event** avec agenda et inscriptions.
- **Backoffice client** : le client gère ses sessions/pages principales sans passer par nous.

**État réévalué (2026-07-19) : chantier global ~35 %** — le repo `back-office-event` porte déjà la
billetterie publique one-page, l'agenda 2 jours, le back-office client, la synchronisation
Attendee, l'inscription publique session et les jauges live. En revanche, son accès admin est
binaire : tout utilisateur authentifié obtient les mêmes droits. Le RBAC demandé par le CDC reste à livrer.

> Nuance : le socle MVP LFD est avancé, mais le chantier global est réévalué à ~35 % tant que
> le RBAC, l'environnement cible, la finition client, le durcissement et le raccord billet/email ne sont
> pas fermés. Ce n'est pas une plateforme billetterie générique post-event avec offres, paiement,
> builder multi-event avancé et self-service complet.

**Dépendances bloquantes :** la finalisation attend surtout la fin de **H** (inscriptions par
session : lien public, capacité/waitlist), **J** (capacité live : statut plein/fermé temps réel)
et **C2.1/B** (billet PDF + email après inscription).

| Action                                                     | Temps estimé                                      |
| ---------------------------------------------------------- | ------------------------------------------------- |
| Socle MVP LFD + backoffice client (en cours)                | — (déjà entamé)                                   |
| RBAC LFD : administrateur/opérateur/lecteur                  | **~2,5–5 j**, détail dans [BIL](./BIL-billetterie/README.md) |
| Dashboard salle/créneau + export nominatif contrôlé          | **~0,5–1 j net** après contrat J-ENTREES                       |
| Builder / gestion des landing pages avancées                | post-event / à re-cadrer                          |
| Branchement sur H (sessions publiques + capacité/waitlist) | après H                                           |
| Branchement sur J (statut live plein/fermé)                | après J                                           |
| Branchement sur C2.1/B (billet PDF + email)                | après B0/C2.1                                     |
| **Sous-total BIL MVP LFD réévalué**                         | **~7–11 j-dev**, dont ~3 j consommés               |

> ⚠️ **Impact capacité :** BIL était sous-estimé dans le périmètre initial. Il est désormais
> suivi comme chantier produit à part entière dans [03-suivi-chantiers.md](./03-suivi-chantiers.md).

---

## Récap des temps — périmètre du mois (fourchettes prudentes)

| Chantier                                                                                  | Temps dev estimé                             | Temps calendaire en plus                              |
| ----------------------------------------------------------------------------------------- | -------------------------------------------- | ----------------------------------------------------- |
| **A** — Refonte propre §7 _(priorité n°1)_                                                | **~1 – 2 jours** _(prudent)_                 | —                                                     |
| **I** — ⚡ Levier débit n°1 (compteur capacité + tx courte)                               | **~1,5 jours**                               | —                                                     |
| **0-CI** — CI/CD **minimal**                                                              | **~2 jours**                                 | —                                                     |
| **0-MON** — Monitoring **minimal**                                                        | **~2 jours**                                 | comptes outils (uptime/Sentry)                        |
| **B0** — Email → billet PDF (**POC**, Cloud Run déjà fait)                                | **~½ j**                                     | service Cloud Run (déjà déployé)                      |
| **B1** — Email → billet PDF (**durcissement**)                                            | **~1,5 – 2 j**                               | (~< 1 – 12 € compute)                                 |
| **C** — Migration ESP                                                                     | **~2 – 3 jours**                             | propagation DNS + **warm-up (1-2 sem)**               |
| **D** — Sécurité QR HMAC                                                                  | **~1 – 2 jours**                             | — (coordination 2 repos clients)                      |
| ~~**E** — Backup auto (MVP) + PITR léger~~                                                | ~~**~1,5 – 2 jours**~~                       | ✅ **Backup MVP terminé** (PITR = §3-F, hors scope E) |
| **H** — Inscriptions par session + maximum 2 confirmées/jour                              | **~6,5 – 10,5 jours**                        | —                                                     |
| **J** — Capacité live forte charge + J-ENTREES                                            | **~5 – 9 jours**                             | — (Redis déjà présent)                                |
| **BIL** — Plateforme billetterie LFD (landing + RBAC + dashboard entrées)                 | **~7–11 j-dev** (chantier global ~35 %)      | dépend de H + J + C2.1/B                              |
| **K** — Résilience event (checklist protections : saturation/disque/perte/recovery testé) | **~½ – 1 jour** _(surtout vérif + runbooks)_ | —                                                     |
| **M** — Scan mobile et recette terrain                                                    | **~5,5–9 j-dev**                             | multi-appareils + répétition J-10 et J-4                   |
| **N** — Architecture event-ready (coordination I/J/B/C2.1/T/K + N-ANTI)                   | **~10–17,5 j bruts**                         | ne pas doubler les lots déjà distribués                |
| **O** — Audit métier et sécurité LFD                                                     | **~5–8,5 j bruts**                           | hooks partagés BIL/T/K ; avant k6 final                 |
| **F** — API stateless, multi-instance et HA                                               | 🔵 **15–27 j-dev**                           | ~4–7 semaines calendaires, audit post-event puis GCP   |
| **L** — Architecture event-driven globale                                                 | 🔵 **27–46 j-dev**                           | ~2–4 mois calendaires, livraison incrémentale          |
| **P** — Plateforme de journalisation globale                                              | 🔵 **15–25 j-dev après L0/L1**               | 22–36 j autonome ; entièrement post-event              |
| **G** — Wallet (V2)                                                                       | ~1 semaine _(hors event)_                    | validation Apple                                      |
| **Reste event brut au 19/07**                                                            | **~32 – 48,5 jours-dev**                     | suivi vivant ; J-ENTREES recoupe H/BIL/T/K             |

> ⚠️ **Chevauchement à arbitrer :** le chantier **J** (capacité live forte charge) recoupe en partie
> **H** (inscriptions par session : capacité/waitlist) et **I** (compteur capacité). Ne pas double-compter :
> une fois H/I cadrés, une partie de J est déjà couverte. J = le **sur-ensemble forte charge** (portier
> Redis anti-survente + WebSocket live + pic combiné insc/check-in), à livrer pour l'event.

### Chantier N — Architecture event-ready LFD 2026 🔴 Haute (event)

> **📎 Cadrage :** [N — architecture event-ready](./N-architecture-event-ready/README.md)

Chantier coordinateur event : chemins inscription/check-in synchrones et atomiques, effets secondaires
BullMQ, workers PDF/email séparables, idempotence, réconciliation, tests de charge combinés,
métriques et runbooks. Il consolide I/J/B/C2.1/T/K/O sans doubler leur estimation. Il inclut désormais
[N-ANTI](./N-architecture-event-ready/N-ANTI-protection-anti-bot.md) : Turnstile validé serveur,
rate limit progressif, idempotence et correction du risque de quota unique derrière le proxy PHP.
Charge brute coordonnée : **~10–17,5 j**, dont une part importante est déjà portée par les chantiers liés.

**Impression/PrintNode est explicitement hors scope LFD** et reste dans les trajectoires post-event F/L.

### Chantier O — Audit métier et sécurité LFD 🔴 Haute (CDC)

> **📎 Cadrage :** [O — audit LFD](./O-audit-lfd/README.md)

O couvre avant l'événement la journalisation des accès et actions back-office : authentification,
refus, modifications d'inscription, rôles, ouverture/fermeture, jauges, horaires et exports. Le socle
PostgreSQL `audit_logs` est conservé, mais les événements critiques deviennent explicites et
versionnés via `AuditEventV1`, avec auteur/rôle snapshot, cible, résultat et avant/après minimisé.
`O-BIL` est un lot d'intégration partagé avec Corentin, pas un chantier supplémentaire.

Le k6 ciblé L9.b est exécuté avant O. Le k6 final est rejoué avec O activé pour vérifier coût et
complétude dans la configuration réellement livrée.

### Chantier L — Architecture event-driven Attendee 🔵 Post-event

> **📎 Cadrage :** [L — architecture event-driven](./L-architecture-event-driven/README.md)

Trajectoire globale après LFD : transactional outbox/inbox, événements métier versionnés, consommateurs
idempotents, replay/DLQ, traçabilité et migration progressive des flux inscription, check-in, billet,
email, impression, invitations, scoring, exports, intégrations et webhooks. Ce chantier est distinct de F :
F rend les instances interchangeables ; L fiabilise et découple la propagation des faits métier.

### Chantier P — Plateforme de journalisation Attendee 🔵 Post-event

> **📎 Cadrage :** [P — plateforme de journalisation](./P-plateforme-journalisation/README.md)

P réutilise `AuditEventV1` de O et L0/L1 pour brancher l'audit sur une outbox durable, construire les
projections, l'archive, la rétention et le viewer org-scoped. Il consolide aussi les logs techniques
dans un pipeline séparé. Aucun choix MongoDB/OpenSearch/ClickHouse n'est pris avant volumétrie et
benchmark ; P ne recrée ni le format O ni l'outbox L.

> **⚡ Lecture capacité au 19/07 :** ~32–48,5 jours-dev bruts restants sur
> **2 devs × ~15 j ouvrés = ~30 j-dev de capacité restante**. Avec O, la fourchette haute dépasse désormais
> la capacité ; une partie recoupe BIL/T/K, mais **le buffer doit être considéré nul tant que O0 n'a
> pas recalé l'incrément net**.
> Conditions impératives : **tenir le descope** (HA → GCP, CD complet → post-event, wallet → V2),
> traiter **H phase 2 (front) en V1 réduite**, et **prioriser dur** (L9b/L9a/L9.1 + B0 + J d'abord).
> Voir le **planning daté** ci-dessous.

---

## 4. Reste à faire — synthèse priorisée (tous chantiers)

> **Contexte :** échéance **1 mois**, **2 devs**. Périmètre **descopé** pour tenir le délai.

| Priorité | Chantier                                                                | Décision                                                                                                  |
| -------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 🔴 P0    | **A** — Refonte propre §7                                               | **Priorité n°1** — débloque le déploiement propre du reste                                                |
| 🔴 P0    | **C1–C3** — Mailgun + warm-up                                           | C1/C2 faits ; poursuivre C3 sans utiliser le fallback SMTP pour les lots                                  |
| 🔴 P0/P1 | **I** — ⚡ Levier débit n°1 (compteur capacité dénormalisé + tx courte) | **Le seul levier qui lève le plafond ~30/s** — juste après A, avant le load test S4                       |
| 🔴 P1    | **0-MON** — Monitoring minimal (uptime + alerte + Sentry)               | Gardé (réduit)                                                                                            |
| 🔴 P1    | **0-CI** — CI/CD **minimaliste** (build+test PR + deploy staging)       | Gardé (réduit, **sans CD/rollback auto**)                                                                 |
| ✅ —     | **E** — Backup auto MVP                                                 | ✅ **Terminé** — [PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15)                          |
| 🔴 P1    | **H** — Inscriptions par session                                        | Besoin fonctionnel LFD ; finir la V1 et la limite de deux activités confirmées/jour                      |
| 🔸 P1    | **D** — Sécurité QR HMAC                                                | Gardé (simple, données MEAE)                                                                               |
| 🔴 P1    | **J** — Capacité live + J-ENTREES                                  | finir L9.1, dashboard salle/créneau et export présent/absent avant répétition                              |
| 🟠 P1/P2 | **BIL** — Plateforme billetterie LFD (landing + RBAC + entrées)      | ~35 % ; RBAC et vue client CDC à livrer avec H, J et C2.1/B                                               |
| 🟠 P1    | **B/B2** — Email → billet PDF + fallback Puppeteer borné                | Finir B1/C2.1 ; B2 partage le worker PDF et protège l'API en panne Gotenberg                              |
| 🟠 P1    | **C4** — Fallback SMTP résilient                                        | Mode manuel d'abord, transactionnel critique uniquement, quota OVH à confirmer                           |
| 🔴 P1    | **N** — Architecture event-ready                                        | **Coordonne les garanties sync/async, la charge et les runbooks sans doubler I/J/B/C2.1/T/K/O**           |
| 🔴 P1    | **O** — Audit métier et sécurité LFD                                    | **Exigence CDC ; socle réutilisable, O-BIL partagé, à fermer avant le k6 final**                           |
| 🔴 P1    | **M** — Scan mobile et recette terrain                                  | **Gate jour J : corrections offline/multi-scanner, parc réel et répétition GO/NO-GO**                      |
| 🔵 —     | **F** — API stateless, multi-instance et HA                             | **Reporté post-event** ; audit applicatif avant activation du clustering/GCP                              |
| 🔵 —     | **L** — Architecture event-driven globale                               | **Reporté post-event** ; outbox/inbox et migration progressive des flux                                   |
| 🔵 —     | **P** — Plateforme de journalisation                                    | **Reportée post-event** ; réutilise O + L0/L1, aucun sink décidé d'avance                                 |
| ⚪ V2    | **G** — Wallet Apple + Google                                           | Hors périmètre event                                                                                      |

> **🔵 Migration GCP** : absorbe la HA + le PITR managés. La continuité lourde n'est donc **pas** à
> construire à la main ce mois-ci — pour l'event, le filet = **backup MVP (+ PITR léger)**.

---

## Planning 1 mois — 2 devs (proposition datée)

> Hypothèse : 2 devs, ~4 semaines. Estimations **prudentes** (réalité incluse, pas best-case IA).
> ⚠️ **À lancer dès J1 (calendaire, pas du dev) :** création compte ESP + configuration DNS
> (SPF/DKIM/DMARC) + **warm-up** — sinon les ~12 400 emails risquent le spam.

| Semaine | **Dev 1 (back / infra)**                                                                                           | **Dev 2 (CI / clients / sécu)**                                                                                    |
| ------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| **S1**  | **A — Refonte propre** (rejouer les leviers, redéployer proprement) · lancer setup ESP (DNS)                       | **0-CI minimal** (build+test sur PR + deploy staging) · **0-MON** uptime + alerte                                  |
| **S2**  | **I — Levier débit n°1** (compteur capacité + tx courte, mesure avant/après) · **E — Backup MVP** + **PITR léger** | **D — QR HMAC** (back + mobile + print-client) · finaliser CI minimal · monitoring back (Sentry + métriques CPU)   |
| **S3**  | **C — Intégration ESP** derrière nodemailer + test envoi en masse                                                  | **B — Email → billet PDF** : déployer **Gotenberg sur Cloud Run** + brancher le rendu + joindre PDF + lien secours |
| **S4**  | **Load test de validation** sur prod réelle (**+ scénario badge / Cloud Run**) · checklist event · **buffer**      | Tests bout-en-bout (scan QR signé, email+PDF, fallback Puppeteer) · runbook event · **buffer**                     |

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
- ✅ **Inscriptions par session** (lien public dédié, capacité + waitlist, contrôle organisateur) + refonte front sessions _(front phase 2 : V1 réduite selon buffer)_
- 🟡 **Plateforme billetterie** (landing pages + backoffice client) — livrée **dans la mesure où H et J sont clos à temps** (sinon : socle + backoffice, branchements sessions/live en fast-follow)

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

- **Inscription plus rapide sous charge** : les transactions allégées **L9b** (session LFD) et **L9a**
  (event classique) sortent les `COUNT`/verrous larges du hot path → moins de contention sur les
  inscriptions publiques.
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
