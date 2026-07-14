# Workstream — LFD 2026 : chaîne de livraison (email · billet · PDF · sessions)

> **Statut :** EN COURS — cadrage fait, chantiers à dérouler.
> **Priorité :** 🔴 Haute (contrat MEAE, échéance 4–5 sept 2026)
> **Repo principal :** `attendee-ems-back` (+ `attendee-ems-front` pour les sessions)
> **Workstream infra/perf lié** (scaling + continuité) : [infra-scaling-pca](../infra-scaling-pca/README.md).

---

## Objet

Tout ce qui touche la **livraison au participant** pour LFD 2026 : email de confirmation,
billet + QR, rendu PDF/badge, et la refonte **inscriptions par session**. Le volet
**perf/scaling + continuité** vit dans son propre workstream (lien ci-dessus).

## Le plan maître

➡️ **[00-plan-action.md](./00-plan-action.md)** — plan d'action consolidé (tous les chantiers
A→I, priorités, planning). C'est le point d'entrée.

> 🗺️ **Carte stratégique (event vs post-event) :**
> [infra/lfd-2026-strategie-2-chantiers.md](../../../infra/lfd-2026-strategie-2-chantiers.md) —
> sépare la **prépa event** (OVH + leviers + Gotenberg + capacité live) du **post-event**
> (scaling horizontal + migration GCP). Décisions du 08/07.

## Contenu du dossier

À la racine, on garde uniquement le **plan** et le **suivi** ; tout le reste est rangé
dans un **dossier par chantier**.

| Fichier / dossier                                 | Sujet                                                                                           |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| [00-plan-action.md](./00-plan-action.md)          | Plan d'action consolidé LFD 2026 (tous chantiers)                                               |
| [03-suivi-chantiers.md](./03-suivi-chantiers.md)  | **📊 Tableau de bord vivant** : avancement % / temps / jalons fin de mois — MAJ chaque vendredi |
| [rapports/](./rapports/2026-W28-rapport-hebdo.md) | Rapports hebdo manager (fait / décisions / qui a fait quoi)                                     |

### Dossiers par chantier

Chaque chantier a son dossier (lettre + nom parlant) avec un `README.md` d'entrée.

| Dossier                                                       | Chantier(s)                                                        |
| ------------------------------------------------------------- | ------------------------------------------------------------------ |
| [A-I-leviers/](./A-I-leviers/README.md)                       | A (refonte propre) + I (levier débit) — inclut le suivi leviers    |
| [0-fondations/](./0-fondations/README.md)                     | 0-CI (CI/CD) + 0-MON (monitoring/alerting)                         |
| [B-email-billet-pdf/](./B-email-billet-pdf/README.md)         | B0/B1 — chaîne email → billet PDF (Gotenberg, diagnostic, lib PDF) |
| [C-migration-esp/](./C-migration-esp/README.md)               | C — migration ESP (délivrabilité ~12 400 envois)                   |
| [D-securite-qr/](./D-securite-qr/README.md)                   | D — sécurité QR (HMAC)                                             |
| [E-sauvegarde-db/](./E-sauvegarde-db/README.md)               | E — sauvegarde DB auto ✅                                          |
| [H-inscriptions-session/](./H-inscriptions-session/README.md) | H — inscriptions par session                                       |
| [J-capacite-live/](./J-capacite-live/README.md)               | J — capacité live forte charge                                     |
| [M-appli-mobile-lfd2026/](./M-appli-mobile-lfd2026/README.md) | M — stabilisation appli mobile pour l'event                        |
| [BIL-billetterie/](./BIL-billetterie/README.md)               | BIL — plateforme billetterie                                       |
| [K-resilience-event/](./K-resilience-event/README.md)         | K — résilience event (+ décision « tenir sans GCP »)               |
| [F-continuite-ha/](./F-continuite-ha/README.md)               | F — continuité HA (reporté → GCP)                                  |
| [G-wallet/](./G-wallet/README.md)                             | G — wallet Apple + Google                                          |

## Chantiers portés ici (cf. plan maître)

- **B** — Chaîne email → billet PDF (rendu Gotenberg/Cloud Run).
- **C** — Migration **Mailgun** (EU, IP partagée) — délivrabilité de masse (C1 prépa+warm-up / C2 intégration).
- **D** — Sécurité QR (signature HMAC + durcir route publique).
- **H** — Inscriptions par session → [workstream dédié](../../a-faire/sessions-inscriptions-lfd2026/README.md).

> Les chantiers **perf (I)** et **continuité/HA (F)** sont pilotés dans
> [infra-scaling-pca](../infra-scaling-pca/README.md).
