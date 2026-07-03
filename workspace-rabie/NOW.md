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
➡️ Reproduire et instrumenter l'éjection au hard restart — voir [workstreams/mobile-stabilization/01-eject-hard-restart.md](../workstreams/fait/mobile-eject-socket-resilient-delta-async/01-eject-hard-restart.md).

---

## ⚡ Travail parallèle en cours — Scaling API LFD 2026 (contrat MEAE)

```
Workstream: Scaling API & charge LFD 2026
App:        attendee-ems-back
Status:     Diagnostic terminé, optimisations partielles livrées
Priority:   High (contrat client, 4–5 sept 2026)
Session:    2026-06-25
```

**Reprise → lire d'abord** : [infra/lfd-2026-session-handoff-2026-06-25.md](../infra/lfd-2026-session-handoff-2026-06-25.md)
et le workstream [api-scaling-lfd2026](../workstreams/en-cours/api-scaling-lfd2026/README.md).

- ✅ Diagnostic prouvé : plafond ≈ **33–37 inscriptions/s** = **sérialisation écriture DB**
  (ni CPU, ni pool, ni nb de process). Couvre le besoin contractuel (3000 en ~90 s).
- ✅ Livré : pool DB, email→BullMQ, transaction allégée, worker séparable (`PROCESS_ROLE`).
- ✅ 3 rapports écrits (résultats / reproduction / client).
- ➡️ **Prochain levier** : optimiser la transaction d'écriture (sortir le COUNT capacité).
- 🚨 **Cluster (Voie B) JAMAIS en prod** sans le workstream clustering.

---

## Fait récemment (2026-06-18)

- ✅ **Socket résilient + delta-sync à la reconnexion** livré ([workstreams/mobile-stabilization/03](../workstreams/fait/mobile-eject-socket-resilient-delta-async/03-socket-resilient-delta-sync.md) §8) : reconnexions infinies, delta-sync auto, indicateur d'état socket.
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
