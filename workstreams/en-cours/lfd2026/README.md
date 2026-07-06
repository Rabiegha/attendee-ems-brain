# Workstream — LFD 2026 : chaîne de livraison (email · billet · PDF · sessions)

> **Statut :** EN COURS — cadrage fait, chantiers à dérouler.
> **Priorité :** 🔴 Haute (contrat MEAE, échéance 4–5 sept 2026)
> **Repo principal :** `attendee-ems-back` (+ `attendee-ems-front` pour les sessions)
> **Workstream infra/perf lié** (scaling + continuité) : [infra-scaling-pca](../../a-faire/infra-scaling-pca/README.md).

---

## Objet

Tout ce qui touche la **livraison au participant** pour LFD 2026 : email de confirmation,
billet + QR, rendu PDF/badge, et la refonte **inscriptions par session**. Le volet
**perf/scaling + continuité** vit dans son propre workstream (lien ci-dessus).

## Le plan maître

➡️ **[00-plan-action.md](./00-plan-action.md)** — plan d'action consolidé (tous les chantiers
A→I, priorités, planning). C'est le point d'entrée.

## Contenu du dossier

| Fichier | Sujet |
|---|---|
| [00-plan-action.md](./00-plan-action.md) | Plan d'action consolidé LFD 2026 (tous chantiers) |
| [diagnostics/email-billet-wallet.md](./diagnostics/email-billet-wallet.md) | Diagnostic chaîne email / billet PDF / wallet vs code réel |
| [decisions/esp-brevo-vs-scaleway.md](./decisions/esp-brevo-vs-scaleway.md) | Choix ESP (délivrabilité ~12 400 envois) |
| [decisions/lib-pdf-badge.md](./decisions/lib-pdf-badge.md) | Choix moteur de rendu PDF/badge |

## Chantiers portés ici (cf. plan maître)

- **B** — Chaîne email → billet PDF (rendu Gotenberg/Cloud Run).
- **C** — Migration ESP (Brevo/Scaleway) — délivrabilité de masse.
- **D** — Sécurité QR (signature HMAC + durcir route publique).
- **H** — Inscriptions par session → [workstream dédié](../../a-faire/sessions-inscriptions-lfd2026/README.md).

> Les chantiers **perf (I)** et **continuité/HA (F)** sont pilotés dans
> [infra-scaling-pca](../../a-faire/infra-scaling-pca/README.md).
