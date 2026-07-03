# Backlog — Context Mesh Improvements

Type: Backlog Item
Initiative: Architecture Modernization
Workstream: Context Mesh
Status: Backlog
Priority: Medium-High

## 1. Why this exists

Le projet Attendee est éclaté en **plusieurs repos** :
- `attendee-ems-back/`
- `attendee-ems-front/`
- `attendee-ems-mobile/`
- `attendee-ems-print-client/`
- `Public-Form-Logger/`
- `attendee-context-hub/` — méta-repo de contexte.
- `ems-brain/` — référent IA.

Chaque repo a son propre dossier `context/` (constraints, decisions, playbooks, architecture). Aujourd'hui, ces contextes :
- ne sont pas systématiquement à jour ;
- se chevauchent ou se contredisent parfois ;
- ne sont pas synchronisés avec `docs/` du back.

L'objectif du **Context Mesh** est de garantir qu'un développeur (ou Copilot) qui ouvre n'importe quel repo trouve le contexte juste, à jour, et cohérent avec les autres repos.

## 2. Current understanding

- `attendee-context-hub/` regroupe déjà des analyses et overviews ([attendee-context-hub/overview/](../../../attendee-context-hub/overview/)).
- Chaque repo a un dossier `context/` avec `architecture/`, `constraints/`, `decisions/`, `playbooks/`.
- Il n'existe pas de mécanisme automatisé de synchronisation.
- `ems-brain/` joue un rôle de référent transverse (à clarifier).
- Le `docs/00-CONTROL-CENTER.md` du back (créé via ce chantier) est le premier point d'entrée stable côté back.

## 3. What is already decided

- Le dossier `context/` reste **scoped par repo** (pas de fusion massive entre repos).
- Le hub `attendee-context-hub/` reste **lecture seule croisée** (overviews, analyses), pas le siège des décisions.
- Les **décisions** atterrissent dans `docs/steering/DECISIONS.md` du repo concerné (côté back en premier).

## 4. What is not decided yet

- Convention de nommage cross-repo (`context/decisions/0001-xxx.md` partout ?).
- Cadence et règle de mise à jour des `context/` (au commit ? au start ?).
- Définition précise du rôle d'`ems-brain` vs `attendee-context-hub`.
- Comment référencer une décision back depuis le front (ID stable ? hash ?).
- Outils d'aide (script de validation, lint markdown, table des liens cassés).

## 5. Why not now

- Focus actuel = Async Architecture.
- Le travail de cartographie en cours (Q&A) **est** une partie du context mesh — il faut le finir avant de poser des règles transverses.
- Pas d'urgence opérationnelle.

## 6. When to revisit

- Après stabilisation du workstream Async Architecture (cadres et conventions prêts à être propagés).
- Quand le contexte d'un repo devient bloquant pour un autre (ex : front qui s'écarte des décisions back).
- En tandem avec toute nouvelle décision majeure cross-repo.

## 7. Related docs

- `attendee-context-hub/` (méta-repo).
- `ems-brain/` (référent IA transverse).
- [../00-CONTROL-CENTER.md](../../../attendee-ems-back/docs/00-CONTROL-CENTER.md)
- [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md)
- Dossiers `context/` de chaque repo Attendee.

## 8. Possible future epics

- **Context mesh conventions** : nommage, format, cadence de mise à jour.
- **Cross-repo decision sync** : mécanisme pour qu'une décision back propage un pointeur dans front/mobile.
- **Context lint** : script qui vérifie liens cassés / docs orphelines.
- **AI-friendly index** : index global lisible par Copilot pour retrouver une décision rapidement.

## 9. Notes for Copilot

- **Ne pas** réorganiser massivement les `context/` de tous les repos depuis ce backlog (rabbit hole : [../rabbit-holes/full-docs-reorganization.md](../../rabbit-holes/a-faire/full-docs-reorganization.md)).
- **Ne pas** dupliquer une décision dans plusieurs repos — préférer un pointeur stable.
- Si une décision cross-repo émerge, l'ajouter dans [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md) du back et créer un pointeur depuis les autres repos.
