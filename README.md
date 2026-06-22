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
| **Piloter** | [NOW.md](NOW.md) | Focus actif + prochaine action + hors-scope | Quand le focus change |
| **Capturer** | [INBOX.md](INBOX.md) | Tout en vrac, 1 ligne, taggé | En continu, réflexe |
| **Ranger** | dossiers ci-dessous | Le détail trié | En session de tri |

### Le geste anti-dispersion

Pendant que je bosse sur un workstream, **dès que je vois un truc** (bug, manque, idée, doute, piège) :
1. J'ouvre [INBOX.md](INBOX.md).
2. J'écris **1 ligne taggée**.
3. Je reviens à mon workstream. **Je ne change pas de focus.**

En fin de session (5 min de tri), je vide l'inbox : chaque ligne part dans le bon dossier.

---

## 2. Les dossiers

| Dossier | Contient | Vient de |
|---|---|---|
| [workstreams/](workstreams/README.md) | Ce sur quoi je bosse (découpable en parties) | décision de piloter un sujet |
| [backlog/](backlog/README.md) | Sujets pour plus tard | inbox `[idea]` |
| [bugs/](bugs/README.md) | Bugs confirmés à suivre | inbox `[bug]` |
| [DEBTS.md](DEBTS.md) | Dettes techniques et sécurité connues | inbox `[debt]` |
| [rabbit-holes/](rabbit-holes/README.md) | Pièges à ne PAS creuser | inbox `[trap]` |
| [learnings/](learnings/README.md) | Ce que j'apprends | inbox `[learn]` |
| [architecture/](architecture/README.md) | Décisions d'archi stables | inbox `[archi]` |

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
| Diagnostic & stabilisation mobile | mobile | **Active** | [workstreams/mobile-stabilization/](workstreams/mobile-stabilization/README.md) |
| Async Architecture | back | Active (repo back) | `attendee-ems-back/docs/workstreams/async-architecture/` |

---

## 5. Vertical vs horizontal — la règle

- **Pilotage = horizontal.** Ce cerveau (`ems-brain/`) pilote toute l'app. Un seul `NOW.md`, un seul `INBOX.md`.
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
workstreams/<nom>/
  README.md          ← objectif, état, prochaine action de CE chantier
  01-<partie>.md
  02-<partie>.md
```
J'avance partie par partie. Le `README.md` du workstream dit toujours où j'en suis dedans.

---

## 8. Règles pour Copilot

1. Lire ce fichier puis [NOW.md](NOW.md) avant tout gros changement.
2. Si le sujet est **hors focus** → l'ajouter à [INBOX.md](INBOX.md), ne pas coder.
3. Ne pas transformer une analyse (bug, rabbit-hole) en correction sans accord explicite.
4. Une décision validée → la consigner (workstream, `architecture/`, ou `context/` du repo).
5. Garder `NOW.md` et `INBOX.md` courts.
# attendee-ems-brain
