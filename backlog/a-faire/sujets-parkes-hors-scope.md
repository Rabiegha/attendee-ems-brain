# Sujets parkés — à reprendre plus tard

> **Origine :** liste « Hors-scope maintenant » de [workspace-rabie/NOW.md](../../workspace-rabie/NOW.md).
> **Statut :** parkés pendant la stabilisation mobile. **Documenter ≠ faire.**
> **Date :** 2026-07-02

Regroupe les 4 sujets identifiés comme hors focus actuel. Chacun pointe vers son détail
quand il existe déjà ailleurs — ne pas dupliquer, compléter la source.

---

## 1. Refonte navigation mobile

- **Quoi :** revoir l'architecture de navigation de l'app mobile (stacks, tabs, flux d'écrans).
- **Pourquoi parké :** gros chantier structurel, risque de régression pendant la phase de
  stabilisation scan/offline. À ne pas mélanger avec les fixes en cours.
- **Détail :** pas encore de fiche dédiée. Code concerné : `attendee-ems-mobile/src/navigation/`.
- **Quand reprendre :** après stabilisation mobile (bugs éjection + scan offline soldés).

---

## 2. Migration de la persistance (MMKV / redux-persist)

- **Quoi :** évaluer le passage de `redux-persist` (AsyncStorage) vers **MMKV** (stockage
  natif rapide) pour la persistance du store mobile.
- **Pourquoi parké :** touche le cœur du store et la réhydratation — plusieurs bugs récents
  (éjection cold-start, flash login, purge au logout) sont liés à la réhydratation. À traiter
  **après** que ces bugs soient stabilisés, pas pendant.
- **Détail :** points techniques liés dans [BACKLOG-TECH.md](./BACKLOG-TECH.md#L71)
  (flash login, purge store au logout). Code : `attendee-ems-mobile/src/store/store.ts`.
- **Quand reprendre :** une fois la logique de réhydratation `currentOrgId` figée et testée.

---

## 3. Sécurité EAS / rotation des clés

- **Quoi :** sortir les secrets en clair de `eas.json` / `.env*`, faire tourner la clé
  PrintNode compromise, migrer vers EAS Secrets.
- **Pourquoi parké ici :** déjà formalisé en **dette**, ce n'est pas un chantier backlog neuf.
- **Détail (source de vérité) :** [DEBTS.md — S-01](../../debts/DEBTS.md) + mémoire repo
  `mobile-eas-notes.md` (§ Sécurité — DETTE). **Ne pas dupliquer ici.**
- **Quand reprendre :** dès qu'une fenêtre le permet (dette Haute, clé exposée sur GitHub).

---

## 4. Workstream back — Async Architecture

- **Quoi :** migration des opérations synchrones vers l'exécution asynchrone (BullMQ → futur
  event bus).
- **Pourquoi ici :** avance **en parallèle**, ce n'est pas le focus mobile — juste un rappel
  qu'il tourne pour éviter d'y basculer par réflexe.
- **Détail (source de vérité) :** workstream [en-cours/async-architecture/](../../workstreams/en-cours/async-architecture/README.md).
- **Quand reprendre :** piloté indépendamment côté back, pas depuis ce backlog.

---

## Règle

- Ne rien implémenter d'ici sans le passer d'abord par un `NOW.md` de workspace.
- Pour les sujets 3 et 4, la **source de vérité** reste `DEBTS.md` / le workstream — cette
  fiche n'est qu'un aiguillage.
