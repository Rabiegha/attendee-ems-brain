# Rapport hebdomadaire — Semaine 29 (vendredi 10 soir → vendredi 17 juillet 2026)

> **Destinataire :** manager · **Équipe :** Rabie + Corentin (2 devs)
> **Contexte :** préparation événement **LFD 2026** (MEAE, ~12 400 inscriptions, 4–5 sept 2026).
> Semaine **2 sur 4** du plan d'action 1 mois — objectif : **livrer le périmètre event fin juillet**.
> **Périmètre réel de travail :** la semaine a commencé vendredi soir 10/07, avec une session de travail prolongée
> jusqu'à environ 4h du matin, puis s'est poursuivie jusqu'au point du vendredi 17/07.
> **Réfs :** [plan d'action](../00-plan-action.md) · [suivi chantiers (%)](../03-suivi-chantiers.md) · [suivi leviers](../A-I-leviers/01-suivi-leviers.md) · [journal W29](../../../workspace-rabie/journal/2026-W29.md)

---

## 1. Résumé exécutif

- ✅ **Chaîne CI/CD validée de bout en bout** : CI + CD staging + CD prod pour le back et le front, en conditions réelles. Un incident prod 502 d'environ 7 min au premier CD prod back a permis d'identifier et corriger deux problèmes critiques (healthcheck BusyBox + reload nginx après recreate API). Les runs suivants sont verts.
- ✅ **Tests event-critical P0 livrés** : health, scan QR signé, permissions event. Le scan QR HMAC est désormais couvert par E2E, pas seulement par tests unitaires.
- ✅ **Chantier H sessions fortement avancé** : E2E H-T1→H-T14 verts, capacité check-in distincte, onglet Sessions refondu, liste inscrits/stats/lien public/capacité/waitlist en staging.
- 🟢 **BIL / back-office événement a pris forme en produit LFD concret** : le repo `back-office-event` n'est plus un simple prototype de page ; il porte une billetterie one-page LFD avec agenda 2 jours, choix de session, formulaire public, back-office client, synchronisation des sessions vers Attendee et jauges live.
- ✅ **Mailgun C1/C2 livrés** : domaine `mail.attendee.fr` configuré et vérifié, transport Mailgun intégré, queue BullMQ email, webhooks signés, persistence `email_events`, endpoint stats.
- 🟡 **C2.1 cadré** : séquencement inscription session → billet PDF → email. Décision actée : la file PHP locale de `back-office-event` peut dépanner court terme, mais la cible durable doit vivre côté Attendee/BullMQ (`ticket.generate` puis `email.send`).
- 🟢 **Warm-up C3 exécuté en réel** : smoke tests validés, tracking HTTPS actif, désinscription testée bout en bout, lot J1 Channel Scope terminé (**100/100 envoyés**).
- 🟡 **Chantier A/I repris proprement** : L3 `directUrl` + L9b `registered_count` mergés et déployés sur staging, CI/CD verts. La mesure k6 L9b reste bloquée faute de session/token de test dédié.
- 🟡 **Réconciliation Git maîtrisée** : la PR `staging -> main` conflictuelle (#34) a été remplacée par une PR propre (#35), CI verte, non mergée sans décision explicite car `main` peut déclencher la chaîne prod.

Avancement global estimé : **~40 %** du périmètre event. Le niveau de livraison a fortement progressé, mais le buffer reste mince : les risques principaux sont maintenant la mesure/tenue du pic d'inscription, le billet PDF, le mobile, C2.1 et la poursuite quotidienne du warm-up Mailgun.

---

## 2. Livrables de la semaine

| Livrable | Chantier | État | Preuve |
| --- | --- | --- | --- |
| **CI/CD back + front complet** : CI PR, build image GHCR, CD staging, CD prod, rollback/healthchecks | 0-CI | ✅ Validé en réel | Workflows back/front + suivi 03 |
| **Corrections CD prod back** : reload nginx après recreate API, healthcheck BusyBox sans `-4`, URL healthcheck `/api/health` | 0-CI | ✅ En prod | Runs CD prod back verts après incident |
| **Corrections CD prod front** : permissions `actions:read`, healthcheck avec SNI via `--resolve`, rollback auto validé | 0-CI | ✅ En prod | Runs CD prod front verts |
| **T1/T2/T3 tests event-critical** : health, QR HMAC, permissions event | T / D | ✅ Mergé staging | PR #18/#19 back |
| **E2E sessions H-T1→H-T14** | H | ✅ Verts | `test/sessions-h.e2e-spec.ts`, PR #22 |
| **Capacité check-in distincte (`checkin_capacity`) + refonte onglet Sessions** | H | 🟢 Sur staging | PR #23/#24/#25 back, PR #12/#13 front |
| **Back-office événement LFD** : page billetterie publique one-page, agenda 2 jours, grille horaires × lieux, popup inscription, preview live | BIL | 🟢 Socle fonctionnel | repo `back-office-event` (`index.php`, `admin/index.php`) |
| **Synchronisation BIL → Attendee** : création/update/delete des sessions Attendee depuis l'agenda local, activation du lien public et stockage `public_token` | BIL / H | 🟢 Branché staging | `back-office-event/admin/save.php`, `admin/attendee.php` |
| **Inscription session autoritative** : le formulaire public appelle `POST /api/public/events/sessions/:sessionToken/register`; Attendee reste source de vérité capacité | BIL / I / J | 🟢 Aligné L9b | `back-office-event/register.php` + L9b PR #33 |
| **Jauges live publiques** : abonnement WebSocket aux `public_token` de session, fallback `status.json`, refresh UI sans surcharger Attendee | BIL / J | 🟢 En place | `back-office-event/index.php` |
| **File billet/email côté BIL** : inscription acceptée → job local `queue.jsonl` pour génération billet/envoi email | BIL / B / C | 🟡 Squelette prêt | `back-office-event/register.php`, `worker.php` |
| **Mailgun C1** : plan acheté, domaine EU `mail.attendee.fr`, SPF/DKIM/DMARC/MX/CNAME tracking, webhooks, variables staging/prod | C1 | ✅ Terminé | Suivi C1/C3 |
| **Mailgun C2** : transport SMTP/Mailgun, queue BullMQ `email-send`, webhooks signés, `email_events`, endpoint stats | C2 | ✅ Mergé staging | PR #20 |
| **C2.1 email + billet session** : décision d'architecture sur le flux post-inscription session, et justification file PHP locale vs Attendee/BullMQ | C2.1 / BIL / B | 🟡 Cadré, à implémenter | [note C2.1](../C-migration-esp/C2-1-email-billet-session/03-note-file-php-vs-attendee-bullmq.md) |
| **Warm-up Mailgun C3** : script d'envoi, diagnostic Mailgun, smoke tests, tracking HTTPS, unsubscribe testé, lot J1 terminé (**100/100**) | C3 | 🟢 J1 terminé | `send-warmup-wave1.js`, `mg-events.js`, suivi C3 |
| **L3 `directUrl` Prisma** | A | 🟣 Sur staging | PR #32 |
| **L9b session `registered_count`** : compteur DB, migration/backfill, capacité session sans `COUNT(*)` hot path, refresh admin/public/remove | A / I / J | 🟣 Sur staging + CI/CD OK | PR #33, merge `44a9ff4`, CD staging OK |
| **Réconciliation `staging -> main` propre** | Git / gouvernance | 🟢 PR prête, non mergée | PR #35 `CLEAN`, CI verte ; #34 fermée |
| **0-MON prod opérationnel** : Netdata durci, alertes disque/CPU/RAM, webhook, Sentry, timer BullMQ, health queues | 0-MON | ✅ Terminé | `chore/monitoring`, commit back `63c2bd6` |
| **Chantier M mobile cadré** | M | 🟡 Démarré | `M-appli-mobile-lfd2026/README.md` |

---

## 3. Décisions prises cette semaine

| Décision | Impact | Réf |
| --- | --- | --- |
| **Tout lancer en CI à chaque PR** plutôt qu'une sélection de tests par release | Moins d'angles morts, CI encore acceptable à notre échelle | `ci-cd-post-event.md` |
| **Release mobile QR HMAC en stand-by** | Une seule release mobile en fin de chantier mobile, plutôt que multiplier les releases terrain | `04-etat-branches.md` |
| **GitHub branch protection reportée post-event** | Pas de changement ownership/secrets/rulesets pendant le rush ; option cible = organisation Attendee après LFD | suivi 0-CI |
| **Mailgun choisi et activé** comme ESP principal | Le risque délivrabilité passe d'un risque d'intégration à un risque de warm-up/calendrier à piloter chaque jour | C1/C3 |
| **Un seul domaine principal warmé : `mail.attendee.fr`** | Réputation concentrée, monitoring plus simple | C3 |
| **Le tracking `opened` reste surveillé, pas encore refactoré** | Si les webhooks deviennent bruyants : queue `email.events` + worker/batch/dédup en levier ultérieur | C3 |
| **L9 séparé en L9b puis L9a** | Priorité au vrai hot path LFD via `back-office-event` : session register avant event register classique | suivi leviers |
| **Attendee devient la source de vérité pour BIL** | Le site billetterie n'incrémente plus localement la capacité ; il délègue l'inscription et les compteurs à Attendee | `back-office-event/register.php` + PR #33 |
| **C2.1 : un endpoint déclenche, les workers exécutent** | `POST /api/public/events/sessions/:sessionToken/register` doit réserver vite, puis enqueue durablement PDF/email côté Attendee ; pas de génération PDF ni d'envoi Mailgun bloquant dans la requête | note C2.1 |
| **La file PHP locale n'est pas la cible long terme** | Évite deux sources de vérité, une durabilité faible, une observabilité limitée et une idempotence fragile | note C2.1 |
| **Ne pas merger #35 sans décision explicite** | Évite un déclenchement prod accidentel après réconciliation `staging -> main` | learning réconciliation |

---

## 4. Problèmes rencontrés / risques

| Problème / risque | Impact | Action / état |
| --- | --- | --- |
| **Incident prod 502 ~7 min au premier CD prod back** | Risque disponibilité prod lors des prochains deploys | ✅ Corrigé : reload nginx intégré + healthcheck corrigé + runs suivants verts |
| **Branch protection indisponible sur plan GitHub Free privé** | Risque humain : PR vers mauvaise branche / merge direct | 🔵 Report post-event ; discipline manuelle + PR dédiée |
| **Divergence `main`/`staging`** | PR `staging -> main` #34 conflictuelle | ✅ Résolu proprement par #35, CI verte, non mergée |
| **k6 L9b non lancé** | Gain L9b non quantifié | Prérequis : session staging dédiée + `public_token` + compte loadtest/JWT valide + éventuellement Docker k6 |
| **Compte `loadtest-admin@staging.invalid` retourne 401** | Bloque création de session test via API | À recréer/réparer côté staging |
| **BIL est désormais intégré à la capacité** | Le chantier avait été sous-estimé au départ ; il reste ~1,5–2 j utiles et peut compresser B0/B1, M ou J si non arbitré | Le suivre comme chantier produit à part entière, pas comme simple dépendance H/J |
| **Back-office-event pointe staging Attendee** | Très utile pour tester, mais à sécuriser avant prod/client | Prévoir bascule env/config propre + secrets/URL prod au moment voulu |
| **C2.1 non implémenté** | L'inscription répond, mais la génération/envoi billet final ne sont pas encore orchestrés durablement côté Attendee | Créer `ticket.generate` + séquencement `email.send` après billet prêt |
| **Worker BIL billet/email encore squelette** | Risque de faire dériver la logique métier dans `queue.jsonl`/`worker.php` | Garder comme adaptateur temporaire seulement ; cible = Attendee/BullMQ |
| **Warm-up Mailgun calendaire** | Délivrabilité des ~12 400 emails dépend des signaux jour par jour | Lot J1 **100/100 terminé** ; analyser bounce/complaint/deferred/unsub avant J2 |
| **Billet PDF B0/B1 pas encore branché** | Pièce jointe PDF / séquencement billet→email encore à livrer | Priorité S30 |
| **Mobile encore à stabiliser** | Risques terrain : offline, perf 20k, recherche, OTA, PrintNode | Chantier M cadré, à exécuter S30/S31 |

---

## 5. Avancement par chantier

| Chantier | État fin S29 | Commentaire |
| --- | --- | --- |
| **A — Refonte propre** | **45 %** | L1/L2/L10 mesuré, L3 et L9b sur staging. Reste k6 L9b, L9a, L9.1, L7/L8. |
| **I — Levier débit** | **35 %** | L9b livré staging ; L9a/L9.1 restent à faire et mesurer. |
| **0-CI** | **98 %** | Chaîne complète rodée ; reste nettoyage/gouvernance post-event. |
| **0-MON** | **100 %** | MON prod opérationnel : Netdata durci, alertes testées, Sentry back/front, `/health/queues`, timer BullMQ et rotation Docker vérifiés. |
| **C1** | **100 %** | Domaine Mailgun prêt. |
| **C2** | **100 %** | Intégration applicative Mailgun mergée staging. |
| **C2.1 — Email + billet session** | **20 %** | Cadrage fait : endpoint session unique, transaction courte, jobs durables `ticket.generate` puis `email.send` côté Attendee/BullMQ. Implémentation restante. |
| **C3** | **40 %** | Warm-up réel lancé et J1 terminé (100/100). Pilotage quotidien à poursuivre selon signaux providers. |
| **D** | **90 %** | Back en prod de facto, E2E QR signé fait ; release mobile différée. |
| **H** | **80 %** | Back/front sessions très avancés ; reste grille mode-aware + page détail + prod finale. |
| **J** | **35 %** | Source attendee + WS live + L9b compteur ; reste portier Redis/cache/load test combiné si nécessaire. |
| **T** | **65 %** | P0 fait ; P1 PDF/auth/queue restant. |
| **M** | **10 %** | Cadrage fait, exécution à lancer. |
| **B0 — Billet PDF / Gotenberg** | **70 %** | Cloud Run Gotenberg privé déployé côté GCP, OIDC cadré, PDF 1 page généré ; reste credential sécurisé, smoke Nest/OVH et branchement back. |
| **BIL — Plateforme billetterie** | **55 %** | Le repo `back-office-event` matérialise le socle LFD : landing/billetterie publique, agenda, admin client, sync Attendee, inscription session, jauges live. Reste durcissement prod, env, billet/email final, finition UX/client. |

---

## 6. Points d'attention manager

1. **Le warm-up Mailgun doit être suivi quotidiennement.** Même si le code est prêt, la délivrabilité est un sujet calendaire : ne pas attendre fin juillet pour découvrir les signaux providers.
2. **La mesure k6 L9b nécessite des données de test propres.** Il faut une session staging dédiée avec `public_token`, capacité contrôlée et un compte admin staging valide.
3. **PR #35 prête mais sensible.** Elle réconcilie `staging` vers `main`; elle est verte, mais doit être mergée seulement avec décision prod explicite.
4. **BIL est un vrai chantier produit, pas juste un front de test.** Il transforme le cahier des charges LFD en parcours client concret, et dépend directement de H/J/L9b/B0/B1.
5. **C2.1 est le chaînon manquant entre BIL, PDF et Mailgun.** Le formulaire peut rester PHP, mais l'orchestration durable du billet/email doit être dans Attendee, sinon support et idempotence deviennent fragiles.
6. **Le billet PDF devient la priorité fonctionnelle S30.** C1/C2 email sont prêts ; il faut maintenant fermer B0/B1/C2.1 pour que l'email transactionnel transporte le bon billet.
7. **Mobile à ne pas repousser trop tard.** Le chantier M porte les risques terrain les plus visibles le jour J : offline, perf, scan, PrintNode, OTA.

---

## 7. Plan recommandé semaine prochaine (S30 — 20 → 24 juillet)

1. **Piloter C3 warm-up J2/J3/J4** : consolider J1, décider 200/400 selon signaux, surveiller bounce/complaint/deferred/unsub/open.
2. **Créer/réparer le contexte k6 L9b** : compte admin staging valide, session publique de test, script k6 session, mesure avant/après.
3. **Coder L9a puis L9.1** si L9b seul ne suffit pas ou pour fermer le plan A/I.
4. **Avancer B0/B1 billet PDF** : Cloud Run/Gotenberg + branchement back + pièce jointe/lien secours.
5. **Implémenter C2.1** : `ticket.generate` côté Attendee, statut billet, puis `email.send` Mailgun seulement quand le billet est prêt.
6. **Stabiliser BIL sur `back-office-event`** : environnement cible, finition admin/client, toggle rapide inscriptions, raccord billet/email final, vérification du parcours cahier des charges complet.
7. **Conserver 0-MON en surveillance** : vérifier les alertes pendant les prochains déploiements et garder la purge legacy côté 0-CI hors MON.
8. **Démarrer M mobile P0** : offline session, perf/sync, bug recherche, OTA, secrets PrintNode.
9. **Décider explicitement #35** : merge prod/main ou attente, selon fenêtre de risque et besoin de livrer staging en prod.

> **Cap fin juillet :** réalisable, mais le tampon dépend surtout de trois verrous : mesure capacité/k6,
> billet PDF, et warm-up email sans signal provider négatif.
