# Chantier T — Tests event-critical (LFD 2026)

> **Statut : 🔴 LISTE À VALIDER — rien n'est codé, rien n'est acté.**
> Ce chantier découle de l'audit Codex sur le chantier 0-CI ([coordination Codex/Claude](../../../workspace-rabie/codex-claude/ci-cd-coordination.md)) :
> les tests actuels suffisent pour faire tourner la CI, mais **ne couvrent pas les chemins qui
> casseraient l'exploitation pendant l'event** (scan QR, permissions, inscription, health réel).

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **T**)
- **Owner :** à répartir — **Statut :** ⚪ à démarrer (validation de la liste d'abord)
- **Lié à :** 0-CI (les tests deviennent le filet de la CI) · D-securite-qr (E2E HMAC) · H (inscriptions)

---

## 1. Objectif (et non-objectif)

**Objectif :** protéger les parcours qui casseraient l'event LFD. Pas de couverture 100 %,
pas de refactor test global. Chaque test ajouté doit répondre à « si ça casse en prod pendant
l'event, est-ce qu'on le voit avant ? ».

**Non-objectif (post-event) :** couverture large des services, tests de composants front,
tests de charge (déjà couverts par le chantier J / load test k6).

## 2. État des lieux (2026-07-13)

### Back ([attendee-ems-back/test/](../../../../attendee-ems-back/test/))

| Fichier | Type | Couvre |
| --- | --- | --- |
| `app.e2e-spec.ts` | E2E API | auth basique, `/users`, `/organizations/me`, création user |
| `auth-refresh.e2e-spec.ts` | E2E API | refresh token, logout |
| `users.service.spec.ts` | unitaire | CRUD UsersService (mocké) |
| `src/**/qr-token.service.spec.ts` | unitaire | sign/verify/isToken HMAC QR |
| `src/**/condition-evaluator.spec.ts` | unitaire | conditions badges |

### Front ([attendee-ems-front/tests/](../../../../attendee-ems-front/tests/))

| Fichier | Type | Couvre |
| --- | --- | --- |
| `e2e/login.spec.ts` | Playwright | smoke login |

### Verdict Codex

> Utile pour la CI, **insuffisant pour l'event** : aucun E2E scan QR / billet déjà scanné /
> token modifié / droits scanner / inscription / health réel avec Postgres+Redis.

## 3. Question posée : et les tests d'intégration ?

Codex a parlé d'unitaires + E2E sans mentionner l'intégration. **Réponse : le point est
couvert, mais il faut le nommer correctement.**

- Dans notre stack NestJS, les « E2E API » proposés (supertest + **vrai Postgres/Redis** via
  `test/setup-e2e.ts`) **sont déjà des tests d'intégration** : ils traversent controller →
  guard → service → Prisma → DB réelle. Pas besoin d'une couche séparée supplémentaire.
- En revanche, 3 points d'intégration critiques ne sont couverts par **aucune** des deux
  catégories de Codex et méritent une décision explicite (voir P1/P2 ci-dessous) :
  1. **Webhooks paiement** (si branché avant l'event) — signature, idempotence, cas erreur.
  2. **Queues BullMQ** (emails/billets) — job enqueued + traité, échec → retry/DLQ.
  3. **Compteur capacité Redis** (levier I / chantier J) — décrément atomique, pas de sur-vente.

**Pyramide retenue (proposition) :** unitaires (logique pure : HMAC, conditions, capacité) →
E2E API = intégration (supertest + Postgres/Redis réels) → smoke front Playwright. **Pas de
4e couche.**

## 4. Liste des tests proposés — ✋ À VALIDER AVANT DE CODER

> ⚠️ **Périmètre (décision 13/07) :** tout ce qui touche aux **sessions / inscriptions / règles
> métier de scan session** est sorti de ce chantier → déplacé dans le **chantier H** (owner
> Corentin), voir [H · tests-sessions.md](../H-inscriptions-session/tests-sessions.md).
> Raison : les règles métier (déjà scanné, event vs session, sessions privées/publiques)
> seront écrites pendant H — impossible de les tester avant. **La V1 du CI/CD est livrée sans.**
> Pas de paiement pour cet event → tests paiement/webhook supprimés.

### P0 — bloquant avant validation du chantier 0-CI

| # | Fichier proposé | Cas couverts | Effort |
| --- | --- | --- | --- |
| T1 | `test/health.e2e-spec.ts` | `/health` → `status=ok` avec Postgres/Redis/migrations réels · présence `version` · dégradé si dépendance KO | ~¼ j |
| T2 | `test/scan-qr.e2e-spec.ts` — **scan principal de l'event** (check-in entrée) | QR valide accepté · **re-scan au même point de contrôle ≠ refus dur** : répond `already_scanned` (2xx) **et l'action est conservée dans les logs** · multi-scan légitime (points de contrôle différents) accepté · billet inconnu/invalide rejeté · **token HMAC falsifié** rejeté (voir note) | ~½ j |
| T3 | `test/permissions-event.e2e-spec.ts` | admin/staff/scanner autorisés selon rôle · non-autorisé refusé · isolation tenant/event | ~½ j |

> **Note T2 — cycle de vie d'un QR :** un même QR est scanné **plusieurs fois légitimement**
> pendant l'event (entrée principale → session 1 → session 2 → checkout → éventuel re-checkin).
> Le multi-scan entre points de contrôle **différents est normal et accepté**. « Déjà scanné »
> = re-scan **au même point de contrôle** (ex. re-présenter le billet à l'entrée alors que le
> check-in entrée est déjà fait) → réponse `already_scanned` + action loggée, pas une erreur.
>
> **« Token HMAC modifié »** = test anti-fraude : le QR contient `v1.<registrationId>.<signature>`
> (chantier D) ; si quelqu'un change l'ID ou la signature, le serveur doit rejeter. Déjà testé
> en **unitaire** ([qr-token.service.spec.ts](../../../../attendee-ems-back/src/common/security/qr-token.service.spec.ts)) —
> T2 vérifie que ce contrôle est **bien branché sur la vraie route de scan** (E2E).
>
> **Périmètre :** le scan couvre 2 contextes — (a) **scan principal event** : testable maintenant,
> c'est le périmètre T2 ci-dessus ; (b) **scan session** (déjà scanné sur la session, pas inscrit
> à l'event, inscrit à l'event mais pas à la session privée, session publique sans inscription,
> droits scanner selon logique métier) : **→ chantier H**. Le cas « user sans droit scanner »
> dépend de la logique métier H — seul le contrôle d'accès générique reste dans T3.

### P1 — fortement recommandé avant l'event

| # | Fichier proposé | Cas couverts | Effort |
| --- | --- | --- | --- |
| T4 | PDF / badges (unitaire + intégration) | génération billet PDF OK (Gotenberg/pipeline B0) · badge généré avec bonnes conditions (complète `condition-evaluator.spec.ts`) · QR présent et valide dans le rendu | ~½ j |
| T5 | Auth critique (compléter `auth-refresh`) | login KO (mauvais mdp, compte inconnu) · session expirée refusée proprement | ~¼ j |
| T6 | `test/email-queue.e2e-spec.ts` (intégration BullMQ) | job billet/email enqueued · échec → retry · queue visible dans `/health/queues` | ~½ j |

### P2 — si le temps le permet / selon avancement des chantiers

| # | Fichier proposé | Cas couverts | Condition |
| --- | --- | --- | --- |
| T7 | Compteur capacité Redis (unitaire + intégration) | décrément atomique · refus à capacité 0 · pas de sur-vente en concurrence | avec chantier I/J |
| T8 | Front Playwright : parcours scan/checkin staff | login staff → scan → résultat affiché | si UI scan web utilisée pendant l'event |

### → Déplacé chantier H (owner Corentin) — voir [tests-sessions.md](../H-inscriptions-session/tests-sessions.md)

- Registration/inscription E2E (ex-T4) · CRUD sessions · ouverture/fermeture d'inscription ·
  règles métier scan session · sessions privées vs publiques · capacité/waitlist.

### ❌ Supprimé

- ~~Webhook paiement~~ — **pas de paiement pour cet event** (décision 13/07).

**Effort total estimé (chantier T seul) : P0 ≈ 1,25 j · P0+P1 ≈ 2,5 j.**

## 4bis. Stratégie d'exécution en CI (décision 13/07)

- **On lance TOUT à chaque PR** — pas de sélection par release/modif : le but est d'attraper
  les régressions imprévues. À notre échelle (~2-5 min), aucun problème de lourdeur.
- Jobs parallèles dans GitHub Actions : `lint` + `unit` + `e2e` → temps total = job le plus lent.
- Si un jour ça dépasse ~10 min : étager (unitaires sur toutes PR · E2E API sur PR vers
  `staging`/`main` · Playwright au merge/nightly). Pas avant.

## 5. Questions ouvertes (à trancher à la validation)

- [ ] Valider la liste P0/P1/P2 ci-dessus (ajouter/retirer des cas) — **avec Claude et Codex**
- [x] ~~T4 (registration) : faisable avant event ou dépend du chantier H ?~~ → **dépend de H, déplacé dans H** (13/07)
- [x] ~~Paiement : le paiement/webhook sera-t-il branché avant l'event ?~~ → **pas de paiement pour cet event, supprimé** (13/07)
- [x] ~~T2 : vérifier le format de réponse exact du « déjà scanné » dans le code actuel~~ → **vérifié 13/07, voir §5bis**
- [ ] T2 : trancher **409 vs 2xx** pour `ALREADY_CHECKED_IN` (reco : garder 409, le mobile offline en dépend)
- [ ] 🔴 T2 : **le re-scan n'est PAS loggé aujourd'hui** → décider où brancher la persistance (AuditLog ?) et dans quel chantier (T ou H)
- [ ] Owner : Rabie, Corentin, ou réparti par chantier (T2 avec D · tests sessions avec H · T7 avec I/J)
- [ ] Les P0 deviennent-ils **bloquants dans la CI** (required status check) dès leur ajout ?

## 5bis. État réel du code scan (vérifié 13/07 dans `registrations.service.ts`)

| Exigence | Code actuel | Verdict |
| --- | --- | --- |
| HMAC branché sur la route de scan | `checkInByCode` → `ScanCodeResolverService` : vérif HMAC stricte, signature invalide → 400, UUID brut refusé | ✅ conforme, mais **aucun E2E** sur la route (unitaire seulement) → c'est le rôle de T2 |
| Déjà scanné | **409 `ConflictException`** `code: ALREADY_CHECKED_IN` + état serveur complet (pour réconciliation **offline mobile**) | ⚠️ pas un 2xx — choix délibéré mobile ; **reco : garder 409**, T2 teste ce contrat |
| Re-scan conservé dans les logs | Exception levée, **rien n'est persisté**. `AuditLog` existe dans Prisma mais jamais appelé ici | 🔴 **non respecté** — dev à ajouter (brancher la persistance sur le chemin `ALREADY_CHECKED_IN`) |
| Multi-scan par point de contrôle | Un seul `checked_in_at` global par inscription + checkout ; pas de scan par session | ⏳ chantier H |

## 6. Definition of done

- P0 (T1–T3) verts en local **et** dans la CI GitHub Actions.
- P0 requis dans la branch protection de `main`/`staging`.
- P1 verts ou explicitement descopés avec trace ici.
- Ce README mis à jour : liste validée, owners, cases cochées.
