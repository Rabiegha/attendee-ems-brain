# Rapport hebdomadaire — Semaine 28 (lundi 6 → vendredi 10 juillet 2026)

> **Destinataire :** manager · **Équipe :** Rabie + Corentin (2 devs)
> **Contexte :** préparation événement **LFD 2026** (MEAE, ~12 400 inscriptions, 4–5 sept 2026).
> Semaine **1 sur 4** du plan d'action 1 mois — objectif : **livrer le périmètre event fin juillet**.
> **Réfs :** [plan d'action](../00-plan-action.md) · [suivi chantiers (%)](../03-suivi-chantiers.md) · [suivi leviers](../A-I-leviers/01-suivi-leviers.md)

---

## Addendum — Fin de journée S29 (vendredi 17/07/2026)

- ✅ **C3 warm-up passé en exécution réelle** (Mailgun prod, domaine `mail.attendee.fr`).
- ✅ **Tracking prêt** : `open`/`click`/`unsubscribes` actifs, protocole `https`, certificat tracking actif.
- ✅ **Smoke tests validés** : envois internes `@choyou.fr` + contrôle Gmail OK.
- ✅ **Parcours désinscription testé bout en bout** :
  - désinscription effective confirmée côté Mailgun (`suppress-unsubscribe`),
  - réinscription testée (suppression de la suppression list),
  - nouvel envoi ensuite `delivered`.
- ✅ **Qualité contenu** : correction du doublon `unsubscribe` / `Se désinscrire` dans le script warm-up ; version envoyée stabilisée.
- 🟡 **J1 warm-up lancé** : lot `100` (`offset=0`, `limit=100`, `delay=30s`) démarré sur base Channel Scope.
- 🟡 **Avancement au moment du point** : `23/100` envoyés, sans échec remonté dans le flux d'exécution.

**Trace technique (IDs de référence)**

- Smoke tests : `20260717152919.9c3785080ad0d063@mail.attendee.fr`, `20260717152923.ce3db60bc149fac0@mail.attendee.fr`, `20260717155000.7af627cb5228c76e@mail.attendee.fr`
- Lot 100 (départ) : `20260717155415.103d741335884afb@mail.attendee.fr`

**Prochaines actions immédiates**

1. Laisser finir le lot 100 et consolider les métriques J1 (`bounce`, `complaint`, `deferred`, `unsubscribed`, `delivered`).
2. Décider J2 (200) uniquement si les signaux restent propres.
3. Mettre à jour le runbook warm-up opérationnel avec le retour J1.

## 1. Résumé exécutif

- ✅ **2 livrables shippés en prod/staging** : le **système de sauvegarde automatique de la DB**
  (chantier E — terminé, en prod) et la **stack staging iso-prod + pgBouncer** (chantier A —
  mergée et mesurée en test de charge).
- 🎫 **Plateforme billetterie lancée** (mardi) : gestion de la billetterie, création de landing pages
  et **backoffice client** — déjà **~40 %**, la finalisation attend surtout les chantiers **H** (sessions)
  et **J** (capacité live).
- 🟢 La **sécurité QR (signature HMAC)** est codée back + mobile et déployée sur staging (chantier D, ~80 %).
- 📞 **Chantier B0 (billet PDF) dérisqué** : **POC Gotenberg réalisé** — rendu badge validé hors VPS,
  comparaison pixel-à-pixel vs Puppeteer prod (accents OK, diff de police à trancher) — puis **call GCP**
  jeudi, dossier de cadrage rédigé et **dump de prod anonymisé (RGPD)** préparé pour le prestataire.
- 🧭 **Journée complète de décisions structurantes** mercredi : tenir l'event **sans migration GCP**,
  résilience event (chantier K), découpage billet PDF (B0/B1) et wallet (G).
- 📊 Avancement global du périmètre event : **~15 %** en fin de semaine 1/4 — conforme, mais
  🔴 **le warm-up ESP (emails) n'est pas lancé** : c'est LE point à traiter lundi (délai calendaire).

---

## 2. Livrables de la semaine

| Livrable                                                                                                                                                                                                                                                                                   | Chantier | État                                                                                   | Preuve                                                                                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backup auto DB** : dump quotidien, rotation GFS, offsite R2 chiffré, restore-test staging, alerting email + rétention optimisée disque VPS                                                                                                                                               | E        | ✅ **En prod**                                                                         | [PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15) (merge 09/07)                                                                   |
| **Stack staging isolée** iso-prod (postgres 16 + pgBouncer + redis + mailpit + api) + pool DB tuné (L1/L2/L10)                                                                                                                                                                             | A        | ✅ Mergée sur `staging` + **mesurée**                                                  | [PR #14](https://github.com/Rabiegha/attendee-ems-back/pull/14) (merge 10/07)                                                                   |
| **Campagne k6 avant/après** sur staging (3 595 + 3 578 inscriptions, 100 % succès, p95 162→149 ms)                                                                                                                                                                                         | A        | ✅ Fait                                                                                | [mesures détaillées](../A-I-leviers/01-suivi-leviers.md#-mesures-k6-avantaprès-2026-07-10-register-baselinejs-50-vus-3-min-event-lfd-published) |
| **Plateforme billetterie** (démarrée mardi) : gestion billetterie + création de landing pages + **backoffice client**                                                                                                                                                                      | BIL      | 🟡 **~40 %** — finalisation en attente des chantiers H (sessions) et J (capacité live) | [suivi chantiers](../03-suivi-chantiers.md)                                                                                                     |
| **QR codes signés HMAC** (back : signature + durcissement routes publiques + vérif sur TOUS les chemins de scan · mobile : support format `v1.<uuid>.<sig>`)                                                                                                                               | D        | 🟢 Sur `staging`                                                                       | PR #16 → `dev`, PR #17 → `staging` (10/07)                                                                                                      |
| **POC Gotenberg (billet/badge PDF)** : rendu badge réel via Gotenberg validé (PDF 1 page, accents OK) + **comparateur pixel-à-pixel** Puppeteer vs Gotenberg (`compare.sh`, diff de police identifiée à trancher) — le chemin « email → billet PDF hors VPS » est **prouvé techniquement** | B0       | ✅ Fait (branche `chore/gotenberg-cloudrun-poc`)                                       | [02-brief-call §2](../B-email-billet-pdf/02-brief-call-gcp-gotenberg.md) + rendus `temp/gotenberg-poc/`                                         |
| **Dossier de cadrage GCP Gotenberg** (chemin critique, planning, RACI) + brief call                                                                                                                                                                                                        | B0       | ✅ Fait                                                                                | [02-brief-call](../B-email-billet-pdf/02-brief-call-gcp-gotenberg.md) + dossier cadrage                                                         |
| **Dump prod anonymisé RGPD** (2,8 Go, 0 PII vérifié, volumes conservés : 70,7k attendees / 132k inscriptions) + mail EN à la team GCP                                                                                                                                                      | B0 / GCP | ✅ Prêt à partager (URL présignée R2)                                                  | scripts `anonymize-dump.sh` + `anonymize.sql`                                                                                                   |

---

## 3. Décisions prises cette semaine

| Décision                                                                                                                                                                                                                                                       | Impact                                                                 | Réf                                                                                                     |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Tenir l'event SANS migration GCP** — le goulot de débit est le code (L9), pas l'infra ; clustering reporté ; gate de réévaluation après L9                                                                                                                   | Descope majeur qui sécurise l'échéance fin juillet                     | [décision](../K-resilience-event/tenir-event-sans-gcp.md)                                               |
| **HA/réplication reportée à la migration GCP** (Cloud SQL apporte HA + PITR managés) — pour l'event, le filet = backup auto (✅ livré)                                                                                                                         | Évite de construire à la main ce que GCP donnera                       | plan §3-F                                                                                               |
| **Nouveau chantier K — résilience event** : checklist finale des protections (saturation, 🔴 disque plein, perte, recovery testé) avant J-7                                                                                                                    | Couvre le risque n°1 souvent oublié (disque plein = API down)          | [ws résilience](../../../workstreams/a-faire/resilience-event-lfd2026/README.md)                        |
| **Découpe chantier B** : B0 (POC billet PDF, Cloud Run déjà déployé) puis B1 (durcissement) ; **G wallet** découpé (harnais now, Google fast-follow, Apple hors chemin critique)                                                                               | Le chemin critique email→billet devient livrable par incréments        | plan §3-B / §3-G                                                                                        |
| **Nouveau levier L9.1** (compteur présence session O(1), colonne PG) + intégration **chantier J** (capacité live forte charge : portier Redis anti-survente + WebSocket)                                                                                       | Lève le plafond mesuré ~30 inscriptions/s pour le pic                  | plan §3-I / ws 02                                                                                       |
| **Workflow git clarifié** (10/07) : branche dédiée → test local → MR vers `staging` (branche d'environnement) → mesures k6 → merge `staging` → `dev`. Branche `staging` **recréée propre** depuis `main`, ancienne archivée (tag `archive/staging-2026-07-10`) | Fin du fourre-tout staging ; diffs revisables ; environnements alignés | [conventions](../A-I-leviers/01-suivi-leviers.md#conventions-à-respecter-rappel-pour-tout-nouveau-chat) |
| **Partage données GCP en Option 2** : dump anonymisé via URL présignée R2 (pas de vraie data tant que non justifié)                                                                                                                                            | Conformité RGPD vis-à-vis du prestataire                               | journal W28                                                                                             |
| **Architecture cible GCP = Stratégie B** visée directement (mono-instance au début, un seul cutover) — pour le post-event                                                                                                                                      | Cadre le call prestataire du 14/07                                     | [ws gcp-migration](../../gcp-migration/README.md)                                                       |

---

## 4. Problèmes rencontrés / risques

| Problème                                                                                        | Impact                                 | Action                                                                                            |
| ----------------------------------------------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 🔴 **Warm-up ESP non lancé** (chantier C) — délai **calendaire** de 1–2 semaines incompressible | Risque spam sur les ~12 400 emails     | **À lancer lundi 13/07 (S29 J1)** : compte ESP + DNS SPF/DKIM/DMARC                               |
| Branche `staging` polluée ressuscitée par un `git pull` (09/07)                                 | Risque de déployer du code non validé  | ✅ Résolu : staging recréée depuis `main`, ancienne archivée · reste : reset du poste de Corentin |
| **P1002 advisory lock** : migrations Prisma incompatibles avec pgBouncer transactionnel au boot | Crash-loop API staging                 | Contournement `RUN_MIGRATIONS=false` · fix propre = levier **L3 `directUrl`** (planifié)          |
| Throttler du chantier D (100 req/min sur `/register`)                                           | Les prochains runs k6 seront throttlés | Prévoir bypass/limite relevée en staging pour les tests de charge                                 |
| Poste Rabie : disque plein 100 % (jeudi matin)                                                  | ~3 h bloquantes                        | ✅ Débloqué                                                                                       |

---

## 5. Semaine prochaine (S29 — 13 → 17 juillet)

1. 🔴 **Lancer le warm-up ESP** (chantier C) — dès lundi, c'est du calendaire.
2. **L9b** (transaction inscription session, hot path LFD) + **L9a** (transaction inscription event classique) + **L9.1** (compteur O(1)) codés **et testés** (non-régression) — cœur du chantier A/I.
3. **Chantier D → prod** : validation E2E (scan QR signé bout-en-bout) puis déploiement.
4. **Call GCP (mar. 14/07)** : validation archi cible B + Cloud Run · partager schéma + dump anonymisé selon retour.
5. Poursuite de la **plateforme billetterie** (landing pages + backoffice client) — prioriser H/J pour la débloquer.
6. Démarrer **0-CI / 0-MON** : CI sur PR + uptime externe + alerte disque.
7. Clore le chantier A (L7, L8, L3 + étapes restantes).

> **Cap fin de mois :** jalons J1 (17/07) / J2 (24/07) / J3 (31/07) → détail dans le
> [suivi des chantiers](../03-suivi-chantiers.md#jalons-de-livraison-fin-de-mois).
