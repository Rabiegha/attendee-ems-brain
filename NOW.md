# NOW — Focus actuel

## Focus actif

```
Workstream: Diagnostic & stabilisation mobile
App:        attendee-ems-mobile
Status:     Active
Priority:   High
```

**Objectif :** Comprendre et corriger les éjections de l'app mobile (hard restart + ouverture du scan).

**Prochaine action :**
➡️ Reproduire et instrumenter l'éjection au hard restart — voir [workstreams/mobile-stabilization/01-eject-hard-restart.md](workstreams/mobile-stabilization/01-eject-hard-restart.md).

---

## Fait récemment (2026-06-18)

- ✅ **Socket résilient + delta-sync à la reconnexion** livré ([workstreams/mobile-stabilization/03](workstreams/mobile-stabilization/03-socket-resilient-delta-sync.md) §8) : reconnexions infinies, delta-sync auto, indicateur d'état socket.
- ✅ **Détection hors-ligne rapide** (~4-6s au lieu de ~30-48s) + **retry silencieux 409 STALE**.
- ✅ Bugs corrigés : crash écran gris (Rules of Hooks dans `ConflictsBanner`), faux conflit STALE sur action + action inverse offline, build EAS débloqué (runtimeVersion).
- ⏳ Reste : valider sur tablette physique (APK preview EAS) les scénarios de coupure réseau.

---

## Objectifs de cette semaine

- [ ] Reproduire l'éjection au hard restart de façon fiable.
- [ ] Reproduire l'éjection à l'ouverture du scan.
- [ ] Isoler la cause racine de chacune (≠ corriger tout de suite).
- [ ] Décider quoi corriger en premier.
- [ ] Tester sur tablette physique les fixes socket/hors-ligne du 2026-06-18.

---

## Hors-scope maintenant

- Refonte navigation mobile.
- Migration de la persistance (MMKV / redux-persist).
- Sécurité EAS / rotation clés (déjà noté en dette, voir mémoire repo).
- Workstreams back (Async Architecture) — en parallèle, pas le focus.

---

## Règle de mise à jour

- Ne pas recréer ce fichier. Le mettre à jour **quand le focus change**.
- Idées non prioritaires → [INBOX.md](INBOX.md).
