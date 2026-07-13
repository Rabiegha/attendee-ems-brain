◊# 03 — Suivi des chantiers LFD 2026 (avancement %, temps, échéance fin juillet)

> **À QUOI SERT CE FICHIER :** tableau de bord **transversal** de TOUS les chantiers du
> [plan d'action](./00-plan-action.md). C'est **le doc de base** pour piloter le mois et
> alimenter le rapport manager hebdo. À mettre à jour **chaque vendredi** (au minimum).
>
> - Le **quoi/pourquoi** de chaque chantier → [00-plan-action.md](./00-plan-action.md)
> - Le détail levier par levier (chantier A) → [01-suivi-leviers.md](./A-I-leviers/01-suivi-leviers.md)
> - Les rapports hebdo manager → [rapports/](./rapports/)
>
> **🎯 Objectif : livrer le périmètre event à la FIN DU MOIS (≈ 31 juillet 2026).**
> Capacité : 2 devs × ~15 j ouvrés restants = **~30 j-dev restants** pour ~20–29 j-dev de reste à faire.
>
> **Dernière mise à jour :** 2026-07-13 (branches `chore/ci-cd` **poussées** back+front · doc de coordination Codex/Claude créée · reste : PR, secrets GitHub, branch protection, E2E event-critical · **nouveau chantier T — tests event-critical créé, liste à valider** · **H phases 0-1 back FAITES** (routes publiques par session + fix `session_choice_ids`) + back-office billetterie câblé sans credentials · ajout feature `registration_opens_at`)

---

## Vue d'ensemble — avancement global

```
Avancement global périmètre event : ▓▓▓▓░░░░░░░░░░░░░░░░ ~22 %
Semaines écoulées :                 1 / 4  (S28 faite · reste S29, S30, S31)
```

> **Méthode (honnête) :** moyenne des avancements pondérée par l'effort estimé (mi-fourchette,
> hors C1 calendaire / BIL non estimé / F-G reportés) → brut ≈ 26 %. Retenu **~22 %** car
> 0-CI/0-MON sont « code complet, branches poussées » mais **pas encore livrés** (PR, secrets, rodage restants) —
> un chantier ne vaut 100 % que rodé en conditions réelles.

| #         | Chantier                                                                                                                                                                                                                       | Owner      | Estimé             | Consommé                | Avancement             | Statut                                                                                                                                                                                                              | Échéance visée  |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | ------------------ | ----------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| **A**     | Refonte propre (rejouer les leviers)                                                                                                                                                                                           | Rabie      | ~1–2 j             | ~1,5 j                  | **30 %** `▓▓▓░░░░░░░`  | 🟡 En cours — L1/L2/L10 mergé+mesuré, reste L9/L9.1/L7/L8/L3                                                                                                                                                        | S29             |
| **I**     | ⚡ Levier débit n°1 (compteur capacité + tx courte)                                                                                                                                                                            | Rabie      | ~1,5 j             | 0                       | **10 %** `▓░░░░░░░░░`  | 🟡 Diagnostiqué + L9.1 cadré, non codé                                                                                                                                                                              | S29             |
| **0-CI**  | CI/CD minimal (build+test PR + deploy staging)                                                                                                                                                                                 | Rabie      | ~2 j               | ~1 j                    | **80 %** `▓▓▓▓▓▓▓▓░░`  | 🟢 **Code complet, branches poussées (13/07)** (CI back+front, image GHCR, `/health` enrichi, script deploy+rollback auto, CD staging+prod blindés) — reste : PR, secrets GitHub, branch protection, rodage staging | S29             |
| **0-MON** | Monitoring minimal (uptime + Sentry + CPU + 🔴 disque)                                                                                                                                                                         | Rabie      | ~2 j               | ~½ j                    | **70 %** `▓▓▓▓▓▓▓░░░`  | 🟢 **Code complet** (Netdata+alertes disque 70/80/85, `/health/queues` BullMQ+cron, rotation logs) — reste : deploy VPS, DSN Sentry, webhook. **Disque VPS nettoyé 76→44 %**                                        | S29–S30         |
| **B0**    | Email → billet PDF (POC Gotenberg/Cloud Run)                                                                                                                                                                                   | Rabie      | ~½ j               | ~1 j (POC+cadrage+call) | **40 %** `▓▓▓▓░░░░░░`  | 🟡 **POC validé** (rendu badge OK, diff police à trancher) + cadrage GCP fait — reste : déploiement Cloud Run + brancher le back                                                                                    | S30             |
| **B1**    | Email → billet PDF (durcissement)                                                                                                                                                                                              | Rabie      | ~1,5–2 j           | 0                       | **0 %** `░░░░░░░░░░`   | ⚪ Après B0                                                                                                                                                                                                         | S30–S31         |
| **C1**    | Migration Mailgun — prépa & warm-up (mail.attendee.fr + SPF/DKIM/DMARC + warm-up)                                                                                                                                              | —          | calendaire 2-4 sem | 0                       | **0 %** `░░░░░░░░░░`   | ⛔ **BLOQUÉ : accès OVH manquant** → DNS + warm-up impossibles (délai calendaire 2–4 sem)                                                                                                                           | dès accès OVH   |
| **C2**    | Migration Mailgun — intégration applicative (API EU + queue BullMQ + webhooks)                                                                                                                                                 | —          | ~2–3 j             | 0                       | **0 %** `░░░░░░░░░░`   | ⏳ ESP décidé (**Mailgun**) — avançable en // (indépendant OVH)                                                                                                                                                     | S29–S30         |
| **D**     | Sécurité QR (HMAC + durcissement routes)                                                                                                                                                                                       | Corentin   | ~1–2 j             | ~1,5 j                  | **80 %** `▓▓▓▓▓▓▓▓░░`  | 🟢 Codé back+mobile, sur `staging` (PR #16/#17) — reste E2E + prod                                                                                                                                                  | S29             |
| **E**     | Sauvegarde DB auto (dump + GFS + R2 + restore-test + alerting)                                                                                                                                                                 | Corentin   | ~1,5–2 j           | ~2 j                    | **100 %** `▓▓▓▓▓▓▓▓▓▓` | ✅ **Terminé — en prod** ([PR #15](https://github.com/Rabiegha/attendee-ems-back/pull/15))                                                                                                                          | ✅ 09/07        |
| **H**     | Inscriptions par session (lien public + capacité/waitlist + front)                                                                                                                                                             | à répartir | ~5,5–8,5 j         | ~2 j                    | **65 %** `▓▓▓▓▓▓▓░░░`  | 🟢 Back phases 0-1 + endpoints liste inscrits/stats + back-office câblé (credentials supprimés) + `registration_opens_at` + **front : config lien public/ouverture/capacité/waitlist + liste inscrits + stats dans l'onglet Sessions** · reste front (grille mode-aware + page détail dédiée) + tests + déploiement (PR fin de chantier)                                                                                                                                                           | S29→S31         |
| **J**     | Capacité live forte charge (portier Redis + WS + pic combiné)                                                                                                                                                                  | à répartir | ~4–7 j             | 0                       | **5 %** `▓░░░░░░░░░`   | 🟡 Cadré (ws 02) — recoupe H/I, ne pas double-compter                                                                                                                                                               | S30–S31         |
| **T**     | 🆕 Tests event-critical (E2E/intégration : health, scan QR event, permissions, PDF/badges — [liste à valider](./T-tests-event/README.md) · tests sessions → [chantier H](./H-inscriptions-session/tests-sessions.md))            | à répartir | ~1,25–2,5 j        | 0                       | **0 %** `░░░░░░░░░░`   | ⚪ Créé 13/07 (audit Codex 0-CI) — **liste P0/P1/P2 à valider** · P0 bloquant pour valider 0-CI · pas de paiement cet event · tests sessions déplacés vers H (Corentin)                                              | S29 (P0)        |
| **K**     | Résilience event (checklist protections + runbooks)                                                                                                                                                                            | binôme     | ~½–1 j             | 0                       | **10 %** `▓░░░░░░░░░`  | 🟡 Checklist créée, vérifs à dérouler                                                                                                                                                                               | S31 (avant J-7) |
| **BIL**   | **Plateforme billetterie** : gestion billetterie + création de landing pages + **backoffice client** ([plan §3-BIL](./00-plan-action.md#chantier-bil--plateforme-billetterie-landing-pages--backoffice-client--haute-produit)) | Corentin   | à estimer          | ~3 j                    | **40 %** `▓▓▓▓░░░░░░`  | 🟡 En cours (démarré 07/07) — ⚠️ **dépend de H et J** (sessions/capacité) pour finaliser                                                                                                                            | après H/J       |
| **F**     | HA réplication                                                                                                                                                                                                                 | —          | —                  | —                       | —                      | 🔵 **Reporté → migration GCP** (Cloud SQL HA/PITR natifs)                                                                                                                                                           | post-event      |
| **G**     | Wallet Apple + Google                                                                                                                                                                                                          | —          | ~1 sem             | 0                       | —                      | ⚪ V2 — onboarding stores seul à lancer (calendaire)                                                                                                                                                                | post-event      |

**Légende statut :** ⚪ à démarrer · 🟡 en cours · 🟢 quasi fini (validation restante) · ✅ livré · 🔵 reporté (décision) · 🔴 alerte.

---

## Lecture capacité (honnête)

- **Reste à faire estimé :** ~20–29 j-dev · **Capacité restante :** ~30 j-dev (2 devs × 3 semaines).
- → **Ça passe, mais le buffer est mince** (surtout si H phase 2 front n'est pas réduite en V1).
- ⚠️ **BIL (plateforme billetterie) n'est PAS dans l'estimation du plan** : c'est un chantier
  nouveau qui consomme de la capacité Dev 2 — **à estimer et arbitrer** vs 0-CI/0-MON/D,
  sachant que sa finalisation dépend de H et J.
- **Conditions pour tenir la fin du mois** (rappel des décisions du plan) :
  1. Tenir le **descope** : HA → GCP, CD complet → post-event, wallet → V2, H phase 3 → post-event.
  2. **H phase 2 (front)** en **V1 réduite**.
  3. Prioriser dur : **L9/L9.1 → B0 → H phases 0–1 → J**, le reste suit.
  4. 🔴 **Débloquer l'accès OVH puis lancer le warm-up Mailgun** — c'est du délai **calendaire**, pas du dev :
     ⛔ **sans accès OVH, ni DNS ni warm-up possibles** ; chaque jour de retard repousse la capacité
     d'envoi des ~12 400 emails (warm-up ~2-4 sem). **C2 (intégration Mailgun) avançable en parallèle.**

## Jalons de livraison (fin de mois)

| Jalon                | Date cible      | Contenu                                                                                                                                 |
| -------------------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **J1 — Fondations**  | fin S29 (17/07) | Chantier A clos (tous leviers mergés+mesurés) · D en prod · CI+MON posés · **accès OVH obtenu → warm-up Mailgun (C1) lancé** · C2 en // |
| **J2 — Fonctionnel** | fin S30 (24/07) | B0+B1 (email→billet PDF) · H phases 0–1 (back sessions) · I mesuré                                                                      |
| **J3 — Livraison**   | fin S31 (31/07) | H phase 2 V1 réduite · J (portier+WS) · K checklist verte · **load test de validation** · runbook event                                 |

---

## Historique hebdo (résumé — détail dans [rapports/](./rapports/))

| Semaine           | Fait marquant                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Δ avancement |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| **S28 (6–10/07)** | E ✅ livré en prod · A : L1/L2/L10 mergé + k6 avant/après · D : 80 % (staging) · **BIL plateforme billetterie démarrée (→ 40 %)** · **POC Gotenberg validé** + cadrage GCP + dump anonymisé · décisions structurantes (sans-GCP, workflow git, K, B0/B1, G) · **11/07 : décision ESP = Mailgun (C→C1/C2) · réorg lfd2026 en dossiers chantier · plans 0-CI + 0-MON écrits · backlog mobile restructuré** · **11-12/07 (week-end) : code 0-CI (→80 %) + 0-MON (→70 %) complet · disque VPS 76→44 %** | 0 % → ~22 %  |
