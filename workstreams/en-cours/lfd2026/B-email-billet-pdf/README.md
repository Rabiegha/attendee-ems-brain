# Chantiers B — Chaîne email → billet PDF

> **Regroupe :** **B0** (POC — Cloud Run + Gotenberg → PDF joint, chemin heureux) et **B1**
> (durcissement : auth s2s, fallback lien, séquencement, edge cases, tests). Déport du rendu
> PDF badges/billets hors VPS.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-B)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (lignes **B0** et **B1**)

## Fichiers du dossier

| Fichier                                                            | Sujet                                                                      |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| [02-brief-call-gcp-gotenberg.md](./02-brief-call-gcp-gotenberg.md) | Brief call GCP + résultats POC Gotenberg (comparateur Puppeteer/Gotenberg) |
| [GCP_Gotenberg.md](./GCP_Gotenberg.md)                             | Dossier de cadrage équipe GCP (archi cible, séquence, RACI)                |
| [lib-pdf-badge.md](./lib-pdf-badge.md)                             | Décision — choix moteur de rendu PDF/badge                                 |
| [email-billet-wallet.md](./email-billet-wallet.md)                 | Diagnostic chaîne email / billet PDF / wallet vs code réel                 |
| [gotenberg-poc/](./gotenberg-poc/)                                 | Rendus du POC (puppeteer.png, gotenberg.png, diff.png)                     |
