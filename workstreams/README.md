# Workstreams — Index

Chantiers de travail, toutes apps confondues. Un workstream = 1 dossier avec un `README.md`
(pilote local) + des parties numérotées `01-…`, `02-…`.

## Découpe (3 états)

| Sous-dossier | Contient |
|---|---|
| [fait/](fait/) | Chantiers terminés. |
| [en-cours/](en-cours/) | Chantiers actifs. |
| [a-faire/](a-faire/) | Chantiers cadrés, pas encore démarrés. |

## En cours

| Workstream | App | Priority | Dossier |
|---|---|---|---|
| Fiabilisation des refresh tokens | back + web + mobile + print | High | [en-cours/auth-refresh-token-hardening/](en-cours/auth-refresh-token-hardening/README.md) |
| Performance applicative du staging — après refresh tokens | front + back | High | [en-cours/staging-performance-hardening/](en-cours/staging-performance-hardening/README.md) |
| LFD 2026 — chaîne de livraison (email/billet/PDF/sessions) | back + front | High | [en-cours/lfd2026/](en-cours/lfd2026/README.md) |
| API scaling — clustering multi-worker (Redis présence + adapter) | back | High | [en-cours/api-scaling-clustering/](en-cours/api-scaling-clustering/README.md) |
| Infra Scaling & Plan de Continuité (LFD 2026) | back | High | [en-cours/infra-scaling-pca/](en-cours/infra-scaling-pca/README.md) |

## À faire

| Workstream | App | Dossier |
|---|---|---|
| Async Architecture | back | [a-faire/async-architecture/](a-faire/async-architecture/README.md) |
| Système de contexte : fiabilité & adoption | transversal | [a-faire/context-system/](a-faire/context-system/README.md) |
| Onboarding & Billing Management | back | [a-faire/onboarding-billing-management/](a-faire/onboarding-billing-management/README.md) |
| Inscriptions & refonte des sessions (LFD 2026) | back + front | [a-faire/sessions-inscriptions-lfd2026/](a-faire/sessions-inscriptions-lfd2026/README.md) |

## Fait

| Workstream | App | Statut | Dossier |
|---|---|---|---|
| Mobile eject / socket resilient delta async | mobile | **Fait — pas encore testé** | [fait/mobile-eject-socket-resilient-delta-async/](fait/mobile-eject-socket-resilient-delta-async/README.md) |
