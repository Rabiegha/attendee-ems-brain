# EMS Brain — Cerveau de pilotage

Point d'entrée unique pour piloter **toute l'app** (back, front, mobile, print-client).
À ouvrir avant chaque session de travail.

---

## 1. Comment je travaille (la méthode)

Une seule règle d'or :

> **Je capture vite (1 ligne, sans réfléchir). Je range plus tard (en mode tri).**

Trois niveaux, rien de plus :

| Niveau | Fichier / dossier | Rôle | Quand j'y touche |
|---|---|---|---|
| **Piloter** | `workspace-<moi>/NOW.md` | Focus actif + prochaine action + hors-scope | Quand le focus change |
| **Capturer** | `workspace-<moi>/INBOX.md` | Tout en vrac, 1 ligne, taggé | En continu, réflexe |
| **Ranger** | dossiers partagés ci-dessous | Le détail trié | En session de tri |

> Le **pilotage est personnel** : chaque dev a son dossier `workspace-<prénom>/`
> ([workspace-rabie/](workspace-rabie/README.md), [workspace-corentin/](workspace-corentin/README.md)).
> Les **dossiers de fond** (workstreams, bugs, backlog, debts…) sont **partagés**.

### Le geste anti-dispersion

Pendant que je bosse sur un workstream, **dès que je vois un truc** (bug, manque, idée, doute, piège) :
1. J'ouvre mon `workspace-<moi>/INBOX.md`.
2. J'écris **1 ligne taggée**.
3. Je reviens à mon workstream. **Je ne change pas de focus.**

En fin de session (5 min de tri), je vide l'inbox : chaque ligne part dans le bon dossier.

---

## 2. Les dossiers

| Dossier | Contient | Vient de |
|---|---|---|
| [workspace-rabie/](workspace-rabie/README.md) | Pilotage perso de Rabie (`NOW.md` + `INBOX.md`) | — |
| [workspace-corentin/](workspace-corentin/README.md) | Pilotage perso de Corentin (`NOW.md` + `INBOX.md`) | — |
| [workstreams/](workstreams/README.md) | Ce sur quoi on bosse — découpé **fait / en-cours / a-faire** | décision de piloter un sujet |
| [backlog/](backlog/README.md) | Sujets pour plus tard — **fait / en-cours / a-faire** | inbox `[idea]` |
| [bugs/](bugs/README.md) | Bugs confirmés à suivre — **fait / en-cours / a-faire** | inbox `[bug]` |
| [debts/](debts/README.md) | Dettes techniques et sécurité connues (registre `DEBTS.md`) | inbox `[debt]` |
| [rabbit-holes/](rabbit-holes/README.md) | Pièges à ne PAS creuser — **fait / en-cours / a-faire** | inbox `[trap]` |
| [learnings/](learnings/README.md) | Ce qu'on apprend (pas de découpe) | inbox `[learn]` |
| [architecture/](architecture/README.md) | Décisions d'archi stables (pas de découpe) | inbox `[archi]` |

---

## 3. Tags d'inbox

```
[bug]   un truc cassé
[idea]  une idée / amélioration
[trap]  un piège, un sujet qui va m'aspirer
[learn] un truc appris
[archi] une question ou décision d'architecture
[debt]  une dette technique ou sécurité
```

---

## 4. Workstreams actifs

➡️ [workstreams/README.md](workstreams/README.md) — l'index à jour.

| Workstream | App | Status | Détail |
|---|---|---|---|
| Diagnostic & stabilisation mobile | mobile | **En cours** | [workstreams/en-cours/mobile-stabilization/](workstreams/en-cours/mobile-stabilization/README.md) |
| Système de contexte : fiabilité & adoption | transversal | **En cours** | [workstreams/en-cours/context-system/](workstreams/en-cours/context-system/README.md) |
| Scaling API & charge LFD 2026 | back | **En cours** | [workstreams/en-cours/api-scaling-lfd2026/](workstreams/en-cours/api-scaling-lfd2026/README.md) |
| Inscriptions & refonte des sessions LFD 2026 | back + front | À faire | [workstreams/a-faire/sessions-inscriptions-lfd2026/](workstreams/a-faire/sessions-inscriptions-lfd2026/README.md) |
| Async Architecture | back | **En cours** | [workstreams/en-cours/async-architecture/](workstreams/en-cours/async-architecture/README.md) |

---

## 5. Vertical vs horizontal — la règle

- **Pilotage = horizontal.** Ce cerveau (`ems-brain/`) pilote toute l'app. Chaque dev a son `workspace-<moi>/` avec **un** `NOW.md` et **un** `INBOX.md`.
- **Détail technique = vertical.** Le contexte stable et profond reste **près du code** (dans `*/context/` de chaque repo). Le cerveau **pointe** vers lui, il ne le duplique pas.

---

## 6. `ems-brain/` vs les dossiers `context/`

Ne pas confondre :

- **`ems-brain/`** (ici) = travail en cours, réflexion, ce qui bouge. **Output.**
- **`<repo>/context/`** = règles stables (contraintes, ADR, conventions, archi figée). Ce que l'IA et moi devons respecter. **Input.**

Le seul pont : quand une décision d'un workstream devient une **règle définitive**, elle quitte `ems-brain/` et atterrit dans le `context/` du repo concerné (ou un ADR).

---

## 7. Découper un workstream

Un workstream = 1 dossier :
```
workstreams/en-cours/<nom>/
  README.md          ← objectif, état, prochaine action de CE chantier
  01-<partie>.md
  02-<partie>.md
```
J'avance partie par partie. Le `README.md` du workstream dit toujours où j'en suis dedans.
Quand il est fini, il passe dans `workstreams/fait/`.

---

## 8. Règles pour Copilot

1. Lire ce fichier puis le `NOW.md` du workspace concerné avant tout gros changement.
2. Si le sujet est **hors focus** → l'ajouter à l'`INBOX.md` du workspace, ne pas coder.
3. Ne pas transformer une analyse (bug, rabbit-hole) en correction sans accord explicite.
4. Une décision validée → la consigner (workstream, `architecture/`, ou `context/` du repo).
5. Garder `NOW.md` et `INBOX.md` courts.
