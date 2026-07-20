# Chantiers B — Chaîne email → billet PDF

> **Regroupe :** **B0** (client Cloud Run/Gotenberg), **B1** (branchement, retries et fallback
> Puppeteer compatible) et **B2** (filet Puppeteer asynchrone, isolé et borné). Déport du rendu
> nominal hors VPS sans supprimer la solution locale de secours.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-B)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (lignes **B0**, **B1** et **B2**)

## Fichiers du dossier

| Fichier                                                            | Sujet                                                                      |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| [02-brief-call-gcp-gotenberg.md](./02-brief-call-gcp-gotenberg.md) | Brief call GCP + résultats POC Gotenberg (comparateur Puppeteer/Gotenberg) |
| [GCP_Gotenberg.md](./GCP_Gotenberg.md)                             | Dossier de cadrage équipe GCP (archi cible, séquence, RACI)                |
| [suivi-gcp-gotenberg-auth-cloudrun.md](./suivi-gcp-gotenberg-auth-cloudrun.md) | Suivi Cloud Run privé, auth OIDC, smoke test et branchement back        |
| [B2-fallback-puppeteer-resilient.md](./B2-fallback-puppeteer-resilient.md)     | **B2** — fallback Puppeteer asynchrone, circuit breaker, budgets et DoD |
| [lib-pdf-badge.md](./lib-pdf-badge.md)                             | Décision — choix moteur de rendu PDF/badge                                 |
| [email-billet-wallet.md](./email-billet-wallet.md)                 | Diagnostic chaîne email / billet PDF / wallet vs code réel                 |
| [gotenberg-poc/](./gotenberg-poc/)                                 | Rendus du POC (puppeteer.png, gotenberg.png, diff.png)                     |
