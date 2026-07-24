# 03 — Suivi des chantiers LFD 2026 (avancement %, temps, échéance fin juillet)

> **À QUOI SERT CE FICHIER :** tableau de bord **transversal** de TOUS les chantiers du
> [plan d'action](./00-plan-action.md). C'est **le doc de base** pour piloter le mois et
> alimenter le rapport manager hebdo. À mettre à jour **chaque vendredi** (au minimum).
>
> - Le **quoi/pourquoi** de chaque chantier → [00-plan-action.md](./00-plan-action.md)
> - Le détail levier par levier (chantier A) → [01-suivi-leviers.md](./A-I-leviers/01-suivi-leviers.md)
> - Les rapports hebdo manager → [rapports/](./rapports/)
>
> **🎯 Objectif : livrer le périmètre event à la FIN DU MOIS (≈ 31 juillet 2026).**
> Capacité au 23/07 : 2 devs × ~6 j ouvrés restants = **~12 j-dev restants** pour ~20–34 j-dev
> bruts de reste à faire. Les recouvrements B2/C2.1, O/BIL/T/K et N évitent une partie du
> double-comptage, mais ne ferment pas l'écart : les validations terrain/charge et l'audit O
> imposent désormais un arbitrage explicite avant le 31 juillet.
> (BIL réévalué à ~7–11 j avec RBAC + dashboard entrées ; C2.1 inclus à ~5–7 j avec suivi utilisateur minimal).
>
> **Dernière mise à jour :** 2026-07-24 (**chantier H terminé et recetté sur staging**) :
> PR back [#68](https://github.com/Rabiegha/attendee-ems-back/pull/68) et front
> [#27](https://github.com/Rabiegha/attendee-ems-front/pull/27) mergées, CI/CD verts et backend
> staging confirmé sur `245427cdf678ba4879ecfa6b0c523c05c2ec18f9`.
> En complément des **24/24 E2E H locaux**, la recette API create-only du 24/07 est verte :
> **52/52 contrôles**, événement isolé `H-RECETTE-20260724122829`
> (`dda8c24f-6b79-4156-b985-f73db2d47aab`), aucune suppression, aucun seeder et aucune donnée
> existante modifiée. Sont notamment prouvés : capacité/waitlist, idempotence, ouverture,
> quota 2/jour séquentiel et concurrent, re-scan 409 et dernière place de scan atomique.
> H passe à **100 %** ; la contre-recette sur appareils physiques reste suivie dans M/K et la
> promotion production relève du run de livraison, sans rouvrir le développement H.
> Précédent : 2026-07-23 (**fort delta livré et branches doc réconciliées**) :
> `main` a été intégré en fast-forward dans `Rabie` ; **J-ENTREES** est mergé et déployé sur
> staging (PR back #66/#67 + front #26, CI/CD vert), avec compteurs live, WebSocket, dashboard
> « Présences », export et harnais k6 check-in ; **L8** API/worker est mergé (#55), CI verte
> et actif sur staging, puis son code a été réconcilié dans `main` et déployé via #65 ;
> **C2.1** génère désormais le vrai badge
> PDF et le joint à l'email de session (#61, staging), même si le smoke métier reste à faire ;
> Gotenberg est activé sur staging avec credential persistant et smoke OVH réel vert
> (**271 ms, 8 117 octets**) ; la CI n'est plus bloquée par le billing. Côté C3, 1 300
> newsletters de lots sont confirmées jusqu'au 21/07, l'audit Mailgun a réconcilié les preuves,
> et le lot de 600 du 23/07 a reçu son `GO` et démarré (non compté terminé sans bilan final).
> La refonte mobile session/scan est maintenant commitée sur `main`.
> Précédent : 2026-07-21 (**BIL-RBAC + finitions design**) puis 2026-07-20
> (**L9 livré et mesuré sur staging** : L9a/L9b/L9.1
> mergés, réservation session atomique, 70 inscriptions/s soutenues et choc 250 validés avec
> invariants ; 75/s et choc 350 refusés. B0/B1 Gotenberg mergés et smoke OIDC OVH vert ; L8 codé
> en PR #55 ;
> **B2/C4 cadrés** : fallback Puppeteer à sortir du process API et SMTP OVH à conserver en lane
> bornée, sous quota et anti-doublon). Précédent : MVP C2.1 email session livré, architecture N
> event-ready consolidée et chantiers F/L/P reportés post-event.
> Précédente : 2026-07-18 soir (**C3 warm-up sécurisé et rampe atténuée** : J1=30 internes,
> J2=100, J3=250 ; dimanche repos ; J4=400, J5=600, J6a=900, J6b=600 maximum. Smokes séparés
> une heure avant les lots, validation visuelle + Mailgun, fichiers `GO` humains obligatoires).

---

## Vue d'ensemble — avancement global

```
Avancement global périmètre event : ▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░ ~70 %
Semaines écoulées :                 2 / 4  (S28–S29 faites · S30 en cours · reste S31)
```

> **Méthode (honnête) :** moyenne conservatrice des avancements pondérée par l'effort estimé
> (mi-fourchette, hors C1/C3 calendaire / F-G-L-P reportés et sans recompter N ;
> **BIL ~7–11 j, C2.1 ~5–7 j**). Recalcul du 24/07 : **69,1 %**, arrondi
> conservatoirement à **~70 %**. La hausse depuis ~65 % vient surtout de H désormais terminé,
> de J-ENTREES mesuré à 85 % et de T1–T6 terminé ; le seul passage de T de 65 à 100 %
> représente environ 1 point global. A/I ont livré les leviers structurants et borné le débit d'inscription,
> 0-MON/0-CI sont opérationnels et BIL dispose désormais de son socle RBAC + entrées live.
> Le risque se concentre sur les preuves encore absentes : smoke métier/fallback/charge PDF,
> montée en charge combinée complète, cold start et recette mobile, **O audit CDC**, K/runbooks
> et les métriques de délivrabilité consolidées de C3.

| # | Chantier | Owner | Importance — pourquoi | Estimé | Consommé | Avancement | Statut | Échéance visée |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --- | ------------------ | ------------------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| **A** | Refonte propre (rejouer les leviers) | Rabie | 🔴 P0 — socle des leviers ; conditionne les corrections de performance propres | ~1–2 j | ~4,5 j | **95 %** `▓▓▓▓▓▓▓▓▓▓` | 🟢 L1/L2/L10, L3, **L9a (#39/#42), L9b (#38/#45/#50), L9.1 (#40)** et désormais **L8 (#55)** sont mergés avec CI verte. L8 sépare `PROCESS_ROLE=api/worker`, déploie API et worker avec la même image sur staging et possède un rollback borné ; son code a été réconcilié dans `main` par #65. Campagne inscription : **70/s pendant 2 min = 8 400/8 400**, choc **250/250**, compteurs exacts. Reste la preuve de charge check-in/combinée ; 3 000 simultanées ne sont toujours pas prouvées. | S29–S30 |
| **I** | ⚡ Levier débit n°1 (compteurs capacité + transactions courtes event/session) | Rabie | 🔴 P0 — lève le goulot d'inscription et protège les jauges sous concurrence | ~1,5–2 j | ~4 j | **90 %** `▓▓▓▓▓▓▓▓▓░` | 🟢 Compteurs O(1), transactions courtes et choix session atomique livrés. L9.1 supprime les `COUNT` du scan, garantit une seule admission sur la dernière place et conserve undo/cascades. Preuve inscription : 70/s et 250 simultanées avec réconciliation DB/audit ; limite observée entre 70–75/s et 250–350. Le harnais check-in J est mergé, mais son exécution reste reportée ; idem pour la preuve multi-générateur si le CDC maintient réellement 3 000 simultanées. | S29–S30 |
| **0-CI** | CI/CD minimal (build+test PR + deploy staging) | Rabie | 🔴 P0 immédiat — filet de livraison des chantiers event | ~2 j | ~2 j | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Blocage billing levé et chaîne de nouveau opérationnelle.** Les PR back #55/#61/#65/#66/#67 et front #25/#26 ont exécuté leurs checks ; les derniers pipelines lint/typecheck, tests, build/image et E2E sont verts. Les CD staging back et front du 23/07 sont réussis ; le CD prod back de réconciliation #65 est également vert. La purge legacy et la gouvernance avancée restent hors MVP 0-CI. | ✅ S30 |
| **0-MON** | Monitoring minimal (uptime + Sentry + CPU + 🔴 disque) | Rabie | 🔴 P0 — détecte les pannes et fournit la preuve du SLA pendant LFD | ~2 j | ~1,25 j (⚠️ estimé, pas de pointage précis) | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **MON prod opérationnel (16/07)** : Netdata système durci (port `19999` direct fermé, accès `/netdata/` basic auth), alertes EMS chargées (`ems_disk_*`, CPU/RAM), webhook Discord branché + **alerte disque testée en réel**, Sentry back/front activé et testé, `/health/queues` OK, surveillance BullMQ installée en **timer systemd** toutes les 5 min (crontab absent), rotation Docker vérifiée sur API/nginx/Postgres/Redis. Incident évité/versionné : nginx prod doit aussi joindre `ems-staging-network` pour résoudre les upstreams staging après recreate. Reste hors MON : purge legacy 912 Mo côté 0-CI. | ✅ S29 |
| **B0** | Email → billet PDF (POC Gotenberg/Cloud Run) | Rabie | 🔴 P0 — débloque le billet PDF exigé et le chantier C2.1 | ~½ j | ~1,5 j | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ Client OIDC privé PR #51 + correctif multipart PR #54 mergés/déployés. **Smoke du 23/07 depuis le conteneur OVH avec l'image staging exacte :** anonyme 403, health OIDC OK, PDF 8 117 octets `%PDF` en **271 ms**. Binding IAM et audience prouvés. Credential persistant hors Git, mode `600`, monté read-only sur l'API staging. `min-instances`, digest et limites Cloud Run restent à confirmer avant la charge. | ✅ S30 |
| **B1** | Email → billet PDF (durcissement) | Rabie | 🟠 P1 — fiabilise la génération PDF sous charge et en cas de panne | ~1,5–2 j | ~2,5 j | **95 %** `▓▓▓▓▓▓▓▓▓▓` | 🟢 **PR #53/#54 mergées et code présent dans `main`** : trois rendus PDF, fallback Puppeteer, retries transitoires, métriques et multipart corrigé. Staging exécute `GOTENBERG_ENABLED=true` avec credential hors Git, mode `600` et volume read-only. Smoke du 23/07 depuis le conteneur réellement déployé : auth/health OK, PDF **8 117 octets en 271 ms** ; anonyme refusé en 403. Restent badge LFD visuel réel, fallback provoqué et décision d'activation prod ; Gotenberg prod reste OFF. | S30–S31 |
| **B2** | [Fallback PDF Puppeteer résilient et asynchrone](./B-email-billet-pdf/B2/B2-fallback-puppeteer-resilient.md) | Rabie | 🟠 P1 — empêche le filet local de saturer l'API/VPS pendant une panne Gotenberg | ~2–4 j bruts, **~1–2 j nets après C2.1** | socle L8 partagé | **10 %** `▓░░░░░░░░░` | 🟡 Le prérequis worker séparé L8 est livré et C2.1 réutilise déjà un `Badge` unique avec fallback Gotenberg→Puppeteer. Il manque encore la queue `pdf.generate`, la concurrence Puppeteer 1, le circuit breaker, les budgets/priorités, la publication R2 atomique et les tests de panne réels. Gap découvert : monter le credential Gotenberg read-only dans le worker avant d'y déplacer le rendu. | S30–S31 après B1/C2.1 |
| **C1** | Migration Mailgun — prépa domaine/DNS (mail.attendee.fr + SPF/DKIM/DMARC + Mailgun EU) | Rabie | 🔴 P0 — domaine email authentifié, préalable à la délivrabilité | calendaire 2-4 sem | — | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Setup terminé 15/07** : plan Mailgun acheté, domaine EU `mail.attendee.fr` créé/vérifié, DNS OVH publiés (SPF/DKIM/DMARC/MX/CNAME tracking), clé API rotatée, webhooks créés, variables staging+prod posées, envois test staging/prod OK. Le warm-up réel sort de C1 et passe dans **C3**. | ✅ S29 |
| **C2** | Migration Mailgun — intégration applicative (API EU + queue BullMQ + webhooks) | Rabie | 🔴 P0 — envoi durable/asynchrone et suivi Mailgun | ~2–3 j | ~2 j | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Mergée sur `staging` (PR #20, 14/07)** : abstraction transport (SMTP/Mailgun), queue BullMQ `email-send`, webhook Mailgun signé + persistence `email_events`, endpoint stats, toggles config, idempotency des envois d'inscription | ✅ S29 |
| **C2.1** | Email + billet session — orchestration post-inscription (`pdf.generate` → R2 → `email.send`) | Rabie | 🔴 P0 — billet nominatif + email attendus après chaque inscription | ~5–7 j | ~1,5 j (cadrage + MVP + PDF joint) | **60 %** `▓▓▓▓▓▓░░░░` | 🟢 **Cœur utilisateur codé, testé, mergé (#61) et déployé staging** : après une inscription session confirmée, `SessionRegistrationEmailService` génère/récupère le `Badge`, rend le vrai PDF via Gotenberg/Puppeteer, joint `badge-<registrationId>.pdf`, puis remet l'email à la queue avec la clé d'idempotence `reg:<regId>:session:<sessionId>:approval`. Le flux reste best-effort via `setImmediate` et exige `approval_enabled` + template lié. Restent le smoke métier réel LFD, la génération PDF déplacée dans le worker, R2/persistance et page confirmation avec polling, ainsi que waitlist→confirmé. B2 porte les garde-fous Puppeteer sans double chantier. | S30–S31 |
| **C3** | 🆕 Mailgun — warm-up & délivrabilité opérationnelle (base newsletter Channel Scope + monitoring provider) | Rabie | 🔴 P0 calendrier — réputation d'envoi à construire avant les 12 400 emails | calendaire jusqu'au 14/08 puis stabilisation | — | **65 %** `▓▓▓▓▓▓▓░░░` | 🟡 **État réconcilié au 23/07** : 1 300 newsletters de lots confirmées (100 + 250 + 350 + 600), 1 315 newsletters acceptées en incluant les smokes avant le lot du 23, 1 303 destinataires uniques et aucun doublon de masse connu. Le 22/07 a volontairement fait 0 lot faute de `GO`; les garde-fous ont fonctionné. Nouvelle base de 1 320 uniques, dry-runs 600/720, unsubscribe et double verrou `GO` préparés. Le lot de 600 du 23/07 a démarré à 10h29 après smoke + `GO`, mais reste **non compté terminé** sans bilan final. Reste : consolider delivered/failures/complaints/unsub par provider, Postmaster/suppressions, committer les durcissements scripts et poursuivre la rampe conditionnelle jusqu'à 3 000/jour. Détail → [warm-up-strategie.md](./C-migration-esp/warm-up-strategie.md). | S29→event |
| **C4** | [Fallback email SMTP résilient et sous quota](./C-migration-esp/c4-fallback-smtp-resilient.md) | Rabie | 🟠 P1 — conserve un canal transactionnel de secours sans doublons ni épuisement du relais OVH | ~2–3 j | 0 | **0 %** `░░░░░░░░░░` | ⚪ Cadré après audit : Mailgun reste primaire ; SMTP prod `ssl0.ovh.net` devient une lane séparée réservée aux emails critiques, avec ledger/handoff idempotent, quota minute/jour, circuit breaker, timeout ambigu sans bascule, mode manuel d'abord et statuts `accepted` ≠ `delivered`. Quota contractuel OVH à confirmer avant codage de la valeur. | S30–S31 après C3 |
| **D** | Sécurité QR (HMAC + durcissement routes) | Corentin | 🔴 P0 — empêche la falsification des QR et sécurise les accès | ~1–2 j | ~1,5 j | **90 %** `▓▓▓▓▓▓▓▓▓░` | 🟢 **Back : EN PROD de facto** (merges staging→main du 13/07, 0-CI) · **✅ E2E scan signé FAIT (14/07)** : `test/scan-qr.e2e-spec.ts` (9 cas, dont falsification et UUID brut rejeté) sur branche `chore/monitoring` · mobile : mergé sur `main` (PR #9) mais **⏸️ release mobile STAND-BY** (décision 14/07 : release unique à la fin du chantier mobile à venir → [04-etat-branches §Mobile](./04-etat-branches.md)) · **rétro-compat VÉRIFIÉE dans le code (14/07)** : back strict (UUID brut refusé, by design), 1.1.7 compatible (relaie la chaîne scannée, QR emails re-signés serveur à l'ouverture ; offline = nom absent du feedback mais replay OK) — ⚠️ **seuls les supports figés d'avant le 13/07 (badges imprimés / PDF stockés / images QR cachées) sont rejetés → confirmer qu'aucun ne sera utilisé pour l'event** · reste : poser `QR_HMAC_SECRET` dédié en prod (fallback JWT actif) · release mobile | release mobile ⏸️ |
| **E** | Sauvegarde DB auto (dump + GFS + R2 + restore-test + alerting) | Corentin | 🔴 P0 — permet la récupération des données après un incident | ~1,5–2 j | ~2 j | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Terminé — en prod** ([PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15)) | ✅ 09/07 |
| **H** | Inscriptions par session (lien public + capacité/waitlist + front + limite CDC/jour) | Corentin | 🔴 P0 — cœur métier LFD : choix de sessions, limites et capacité | **~6,5–10,5 j** | ~4–4,5 j | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Terminé, mergé, déployé et recetté sur staging.** 24/24 E2E H locaux + **52/52 contrôles API staging** sur l'événement isolé `H-RECETTE-20260724122829`, sans suppression ni seeder. Capacité/waitlist, idempotence, ouverture programmée, maximum deux `confirmed` par jour `Europe/Paris`, waitlist hors quota, concurrence 2/3, contrat 409 `ALREADY_SESSION_SCANNED`, modes privé/walk-in et dernière place de scan atomique sont prouvés. Front V1 mode-aware livré via #27 ; back via #68, SHA staging `245427c`. La contre-recette physique reste dans M/K et l'audit détaillé dans O, sans double comptage H/L9.1/J. | ✅ S30 |
| **J** | Capacité live forte charge + [J-ENTREES](./J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd) | binôme | 🔴 P0 — couvre les 3 000 requêtes, les entrées live et la surcapacité | **~5–9 j** | inclus H + L9b + ~2 j | **85 %** `▓▓▓▓▓▓▓▓▓░` | 🟠 **J-ENTREES mergé/déployé et première campagne combinée exécutée le 24/07.** Harnais corrigé pour attribuer une registration distincte par itération. Fixtures API create-only, aucune suppression. Paliers 15/10/5 VU : **197/123/67 scans admis, 0 erreur**, compteurs exacts et aucune surcapacité. Palier 5 VU scan vert (p95 307 ms) ; cinq pollers dashboard font 50/50 HTTP 200 mais p95 **432 ms** pour un seuil 300 ms. Reste : diagnostiquer/fermer ce SLO, rejouer plus haut, scénario inscription+scan+dashboard et preuve multi-générateur si les 3 000 simultanées sont maintenues. Portier Redis décidé seulement après mesure. | S30–S31 |
| **T** | 🆕 Tests event-critical (E2E/intégration : health, scan QR event, permissions, PDF/badges — [liste validée](./T-tests-event/README.md) · tests sessions → [chantier H](./H-inscriptions-session/tests-sessions.md)) | binôme | 🔴 P0 — prouve automatiquement les parcours qui bloqueraient l'événement | ~1,25–2,5 j | ~1,5 j | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Terminé côté code et tests.** P0 : T1 health, T2 QR et T3 permissions. P1 : T5 auth **10/10 E2E** ; T4/T6 **15/15 tests ciblés**, smoke HMAC → QR réel → Chromium → PDF **1/1**, et BullMQ sur Redis réel **3/3** (`waiting` → `active` → `completed`, puis 3 retries → `failed` avec erreur terminale unique). Build et lint ciblé verts. O-TEST reste volontairement estimé dans O. | ✅ S31 |
| **M** | 🆕 [Scan mobile et recette terrain LFD](./M-appli-mobile-lfd2026/README.md) — offline, multi-scanner, appareils, UX et répétition | binôme | 🔴 P0 terrain — le scan doit fonctionner offline et avec plusieurs appareils | **~5,5–9 j** | ~2,5 j (audit + M1/M2/M3 + refonte UX) | **55 %** `▓▓▓▓▓▓░░░░` | 🟡 **M1 + M2 (contrat 409 H/L9.1 end-to-end) + M3 + refonte UX scan/check-in livrés et désormais commités sur `main`** (`f5b98ae`, galerie d'états dev `dd3f94d`). Écran session orienté roster/présence, hero progression et actions check-in/out/undo ; ScanScreen avec modes conservés et bottom sheet `X/Y présents`. Typecheck vert. Le pourcentage ne monte pas faute de preuve device : restent cold start offline (M4), build RC natif (M5), répétition terrain et PV (M6/K). | code S30–S31 · terrain J-10/J-4 |
| **K** | Résilience event (checklist protections + runbooks) | binôme | 🔴 P0 exploitation — répétitions, PCA et décision GO/NO-GO | ~½–1 j | 0 | **10 %** `▓░░░░░░░░░` | 🟡 Checklist créée, vérifs à dérouler ; raccord O ajouté pour alerte d'échec audit, croissance `audit_logs`, reprise et preuve pendant le test combiné. | S31 (avant J-7) |
| **BIL** | **Plateforme billetterie** : landing, backoffice client, [RBAC](./BIL-billetterie/README.md#bil-rbac--acces-multi-utilisateurs-demande-par-le-cdc) + [dashboard entrées](./BIL-billetterie/README.md#bil-entrees--vue-opérationnelle-demandée-par-le-cdc) | **Corentin** | 🔴 P0 MVP — interface client, RBAC et dashboard exigés par le CDC | **~7–11 j** | ~6,5 j | **70 %** `▓▓▓▓▓▓▓░░░` | 🟢 **Socle client désormais complet sur trois axes** : landing/back-office, RBAC par grants (`read/operate/manage`) avec UI lecteur/opérateur/admin, et **dashboard J-ENTREES live + export livré sur staging**. Les finitions session/design/upload et l'action opérateur ouvrir/fermer sont livrées ; comptes `ADMIN_LFD` créés. Le raccord badge PDF + email existe également dans C2.1. Restent la matérialisation opérationnelle `OPERATOR_LFD`/`READER_LFD`, les E2E allow/deny PHP + Attendee, le raccord audit O, le smoke métier de bout en bout et la validation CDC. | S30–S31 |
| **N** | [Architecture event-ready LFD](./N-architecture-event-ready/README.md) — sync critique, workers async, anti-bot et validation/runbooks | binôme | 🔴 P0 coordination — garantit une architecture event-ready sans surconstruire | **~10–17,5 j bruts** | réparti dans I/J/B/C2.1/T/K/O + N-ANTI | **35 %** `▓▓▓▓░░░░░░` | 🟡 Chantier coordinateur, sans double-compter les jours : le synchrone critique inscription/check-in est atomique et mesuré, J-ENTREES est live, et L8 sépare maintenant API/worker pour les consumers email/export. Le PDF C2.1 reste encore rendu depuis l'API avant remise à la queue email. Restent [N-ANTI](./N-architecture-event-ready/N-ANTI-protection-anti-bot.md) (Turnstile/rate limit/idempotence), déplacement PDF vers worker, validations combinées et runbooks K/O. **Impression hors scope LFD.** | avant LFD |
| **O** | [Audit métier et sécurité LFD](./O-audit-lfd/README.md) — accès/actions BIL, contrat versionné, avant/après et preuve CDC | binôme | 🔴 P0 CDC — audit des accès/actions et preuve de conformité | **~5–8,5 j bruts** | audit statique ~0,5 j ; hooks partagés BIL/T | **10 %** `▓░░░░░░░░░` | 🟡 Socle `audit_logs`/viewer conservé. Reste `AuditEventV1`, port/adaptateur PostgreSQL, login échec/logout, rôle snapshot, changements inscription/rôle/session/jauge/export, correction du toggle BIL, vraie IP/user-agent/request ID, rétention/accès client, E2E et preuve CDC. `O-BIL` est un lot d'intégration, pas un troisième chantier. **Le k6 L9b ciblé avant O est fait** ; rejouer 70/s et 250 après activation de tous les audits synchrones pour vérifier la régression. | S30–S31 avant k6 final |
| **F** | [API stateless, multi-instance et HA](./F-stateless-multi-instance/README.md) | — | 🔵 Post-event stratégique — stateless/multi-instance/HA ; risque SLA à faire accepter pour LFD | **15–27 j-dev** | 0 | **0 %** `░░░░░░░░░░` | 🔵 **Post-event, ~4–7 semaines calendaires** : F0 audit 2–3 j, séparation rôles 2–4 j, état partagé 3–6 j, multi-instance 3–5 j, validation 2–4 j, GCP/HA 3–5 j. Recalibrage obligatoire après F0. | post-event |
| **L** | [Architecture event-driven Attendee](./L-architecture-event-driven/README.md) | — | 🔵 Post-event stratégique — cible event-driven globale, non bloquante pour LFD | **27–46 j-dev** | 0 | **0 %** `░░░░░░░░░░` | 🔵 **Post-event, ~2–4 mois calendaires** : socle outbox/standards 7–11 j, premier flux de référence total 10–16 j, puis migration incrémentale check-in, impression et autres domaines. Distinct de F ; nouveau broker exclu. | post-event |
| **P** | [Plateforme de journalisation Attendee](./P-plateforme-journalisation/README.md) — audit event-driven et logs techniques séparés | — | 🔵 Post-event stratégique — plateforme d'audit et de logs réutilisable | **15–25 j-dev après L0/L1** | 0 | **0 %** `░░░░░░░░░░` | 🔵 Entièrement post-event. Réutilise `AuditEventV1` de O et l'outbox/inbox de L ; possède projections, sinks, archive, viewer, rétention et pipeline technique. Aucun choix MongoDB avant mesure. Estimation autonome sans L0/L1 : 22–36 j-dev. | post-event |
| **G** | Wallet Apple + Google | — | 🟡 P2/V2 — utile au produit, mais aucun besoin LFD immédiat | ~1 sem | 0 | — | ⚪ V2 — onboarding stores seul à lancer (calendaire) | post-event |

**Légende statut :** ⚪ à démarrer · 🟡 en cours · 🟢 quasi fini (validation restante) · ✅ livré · 🔵 reporté (décision) · 🔴 alerte.

**Légende importance :** 🔴 `P0` = indispensable pour exploiter LFD ou satisfaire une exigence CDC ;
🟠 `P1` = important, mais séquençable ou livrable en version réduite ; 🟡 `P2/V2` = non bloquant pour
LFD ; 🔵 `Post-event stratégique` = important pour Attendee à long terme, explicitement hors du chemin
critique de septembre. `P0 calendrier` signifie que le délai incompressible compte autant que le dev.

**Owner `binôme` :** Rabie et Corentin ont tous les deux contribué au chantier ; un owner nominatif
reste affiché lorsqu'un seul des deux le porte réellement.

---

## Lecture capacité (honnête)

- **Reste à faire brut estimé :** ~20–34 j-dev · **Capacité restante jusqu'au 31/07 :** ~12 j-dev
  (2 devs × ~6 jours ouvrés). O recoupe BIL/T/K et B2 recoupe C2.1/L8, mais même en retirant ces
  doublons la capacité ne couvre plus tout le périmètre annoncé.
- → **Le périmètre complet ne passe plus sans descope ou report explicite.** Le chemin réaliste est
  de fermer les parcours event critiques et leurs preuves ; B2/C4 complets, N-ANTI étendu et les
  raffinements self-service doivent être séquencés après le noyau.
- ✅ **BIL** est à ~70 % sur ~7–11 j : RBAC, landing et dashboard entrées sont livrés. Reste
  indicatif ~2–4 j pour rôles opérationnels, E2E allow/deny, audit O et validation CDC.
- ✅ **C2.1** est à ~60 % sur ~5–7 j : le cœur inscription session → badge PDF joint → email est
  mergé/déployé staging. Reste indicatif ~2–3 j pour smoke métier, worker/R2 et suivi utilisateur
  minimal. Websocket de progression, resend avancé, correction email self-service et vérification
  email restent hors périmètre minimal.
- **Conditions pour tenir la fin du mois** (rappel des décisions du plan) :
  1. Tenir le **descope** : HA → GCP, CD complet → post-event, wallet → V2, H phase 3 → post-event.
  2. **H phase 2 (front)** en **V1 réduite**.
  3. Prioriser désormais : **smokes B1/C2.1 → finitions H + RC/recette M → O/K → charge
     check-in/combinée**. J-ENTREES et L8 sont déjà livrés.
  4. ✅ ~~Débloquer l'accès OVH~~ **obtenu 13/07** → ✅ **C1 Mailgun setup terminé 15/07** (plan acheté,
     domaine/DNS/env/tests OK). Le délai calendaire restant est maintenant **C3 warm-up** :
     chaque jour compte pour la capacité d'envoi des ~12 400 emails (warm-up ~2-4 sem).

## Jalons de livraison (fin de mois)

| Jalon                | Date cible      | Contenu                                                                                                                                                                                                                                                                                                            |
| -------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **J1 — Fondations**  | fin S29 (17/07) | ✅ L1/L2/L10 + L3 + L9a/L9b/L9.1 · D, C1/C2, MON et P0 tests livrés · campagne k6 inscription bornée à 70–75/s et 250–350 simultanées · C3 lancé |
| **J2 — Fonctionnel** | fin S30 (24/07) | 🟢 L8 + CI/CD rétablie · B0/B1 actifs staging et smoke réel vert · C2.1 badge PDF joint à l'email · J-ENTREES back/front live + export · BIL-RBAC · UX mobile commitée. Restent les smokes métier/fallback et l'exécution k6 check-in. |
| **J3 — Livraison**   | fin S31 (31/07) | H V1 réduite · rôles/E2E BIL · RC + recette mobile · O/K minimaux · smokes PDF/email · **load test de validation** et runbook event. B2/C4 complets passent après ce noyau s'ils ne tiennent pas. |

---

## Historique hebdo (résumé — détail dans [rapports/](./rapports/))

| Semaine           | Fait marquant                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Δ avancement |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| **S28 (6–10/07)** | E ✅ livré en prod · A : L1/L2/L10 mergé + k6 avant/après · D : 80 % (staging) · **BIL plateforme billetterie démarrée (→ 40 %)** · **POC Gotenberg validé** + cadrage GCP + dump anonymisé · décisions structurantes (sans-GCP, workflow git, K, B0/B1, G) · **11/07 : décision ESP = Mailgun (C→C1/C2) · réorg lfd2026 en dossiers chantier · plans 0-CI + 0-MON écrits · backlog mobile restructuré** · **11-12/07 (week-end) : code 0-CI (→80 %) + 0-MON (→70 %) complet · disque VPS 76→44 %** | 0 % → ~22 %  |
| **S29 (13–17/07)** | C1 Mailgun/DNS et C2 queue/webhooks ✅ · MON prod ✅ · D HMAC + E2E, T1–T3 et sauvegarde E ✅ · L9a/L9b/L9.1 codés · B0/B1 intégrés · warm-up démarré · socle sessions, BIL et mobile consolidés. | ~22 % → ~50 % |
| **S30 (20–23/07, en cours)** | Mesures L9 finales (70/s, choc 250) · CI/CD débloquée · L8 worker mergé puis réconcilié dans `main` · Gotenberg staging authentifié et smoke 271 ms · badge PDF joint à l'email session · J-ENTREES back/front + k6 livrés staging · BIL-RBAC/dashboard à 70 % · UX mobile commitée · warm-up réconcilié et relancé sous `GO`. | ~50 % → ~65 % |
