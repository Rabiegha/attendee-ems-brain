# Bugs

Bugs **confirmés** à suivre, toutes apps confondues. Vient de l'inbox `[bug]`.

> **Documenter ≠ corriger.** On corrige via un workstream ou un accord explicite.

## Découpe (3 états)

| Sous-dossier           | Contient                            |
| ---------------------- | ----------------------------------- |
| [fait/](fait/)         | Bugs corrigés (archive).            |
| [en-cours/](en-cours/) | Bugs en cours de correction.        |
| [a-faire/](a-faire/)   | Bugs confirmés, pas encore traités. |

## À faire

| Bug                                                        | App           | Lien                                                                                                              |
| ---------------------------------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------- |
| Éjection mobile — rotation refresh token perdue (HTTP 499) | mobile + back | [en-cours/2026-07-01-eject-mobile-refresh-rotation-499](en-cours/2026-07-01-eject-mobile-refresh-rotation-499.md) |
| Revue sécurité CORS origin                                 | back          | [a-faire/cors-origin-security-review](a-faire/cors-origin-security-review.md)                                     |
| Suppression user & membership multi-tenant                 | back          | [a-faire/user-deletion-multitenant-membership](a-faire/user-deletion-multitenant-membership.md)                   |
| Reset mot de passe — invalider les sessions actives ?      | back          | [a-faire/password-reset-session-invalidation](a-faire/password-reset-session-invalidation.md)                     |

## Fait — résolus (historique)

Le détail vit dans les workstreams concernés :

| Bug                                                                       | App        | Où c'est traité                                                                                                                                                                                        |
| ------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **502 prod 25/06 — collision nom de projet Docker Compose** (post-mortem) | back/infra | [fait/2026-06-25-prod-502-collision-compose](fait/2026-06-25-prod-502-collision-compose.md) · garde-fous → [levier A-I](../workstreams/en-cours/lfd2026/A-I-leviers/garde-fous-deploiement-staging.md) |
| Éjection au hard restart                                                  | mobile     | [workstreams/en-cours/mobile-stabilization/01](../workstreams/fait/mobile-eject-socket-resilient-delta-async/01-eject-hard-restart.md)                                                                 |
| Éjection à l'ouverture du scan                                            | mobile     | [workstreams/en-cours/mobile-stabilization/02](../workstreams/fait/mobile-eject-socket-resilient-delta-async/02-eject-on-scan.md)                                                                      |
| Crash écran gris à la reconnexion (check-in + undo offline)               | mobile     | [workstreams/en-cours/mobile-stabilization/03 §8.3](../workstreams/fait/mobile-eject-socket-resilient-delta-async/03-socket-resilient-delta-sync.md) — `ConflictsBanner.tsx` Rules of Hooks            |
| Faux conflit STALE + état incohérent (action + action inverse offline)    | mobile     | [workstreams/en-cours/mobile-stabilization/03 §8.3](../workstreams/fait/mobile-eject-socket-resilient-delta-async/03-socket-resilient-delta-sync.md) — neutralisation mutations opposées               |
| Cache figé après micro-coupure réseau (socket mort)                       | mobile     | [workstreams/en-cours/mobile-stabilization/03 §8.1](../workstreams/fait/mobile-eject-socket-resilient-delta-async/03-socket-resilient-delta-sync.md) — socket résilient + delta-sync                   |
