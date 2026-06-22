# Bugs

Bugs **confirmés** à suivre. Vient de l'inbox `[bug]`.

> Documenter ≠ corriger. On corrige via un workstream ou un accord explicite.

| Bug | App | Status | Lien |
|---|---|---|---|
| Éjection au hard restart | mobile | En investigation | [mobile-stabilization/01](../workstreams/mobile-stabilization/01-eject-hard-restart.md) |
| Éjection à l'ouverture du scan | mobile | En investigation | [mobile-stabilization/02](../workstreams/mobile-stabilization/02-eject-on-scan.md) |
| Crash écran gris à la reconnexion (check-in + undo offline) | mobile | Corrigé 2026-06-18 | [mobile-stabilization/03 §8.3](../workstreams/mobile-stabilization/03-socket-resilient-delta-sync.md) — `ConflictsBanner.tsx` Rules of Hooks |
| Faux conflit STALE + état incohérent (action + action inverse offline) | mobile | Corrigé 2026-06-18 | [mobile-stabilization/03 §8.3](../workstreams/mobile-stabilization/03-socket-resilient-delta-sync.md) — neutralisation mutations opposées |
| Cache figé après micro-coupure réseau (socket mort) | mobile | Corrigé 2026-06-18 | [mobile-stabilization/03 §8.1](../workstreams/mobile-stabilization/03-socket-resilient-delta-sync.md) — socket résilient + delta-sync |
