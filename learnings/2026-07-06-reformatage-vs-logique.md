# Séparer reformatage et logique (Prettier & diffs propres)

> **Date :** 2026-07-06 · **Contexte :** chantier A (refonte propre LFD 2026), étape 1 du
> [00-plan-action.md](../workstreams/en-cours/lfd2026/00-plan-action.md).
> **En une phrase :** empêcher l'éditeur de reformater automatiquement des lignes qu'on n'a
> pas touchées, pour que chaque commit ne montre **que** la vraie logique.

---

## 1. Le principe

Le chantier A impose une règle : **1 levier = 1 commit = 1 mesure**. On veut pouvoir regarder
le diff d'un commit et dire « voilà exactement les N lignes qui font passer l'inscription de
~25/s à ~30/s ».

« Séparer reformatage et logique » = ne jamais mélanger, dans un même commit :

- du **reformatage** (guillemets, virgules finales, indentation, retours à la ligne — cosmétique,
  produit par Prettier/ESLint) ;
- de la **logique** (le vrai changement de comportement).

---

## 2. Le piège concret

On ouvre `src/public/public.service.ts` pour rejouer le levier L9. On change **5 lignes** de
logique. Mais si l'éditeur reformate à la sauvegarde avec des réglages qui diffèrent un peu du
style du fichier, Prettier réécrit aussi **~200 lignes autour**. Le commit devient :

```
 public.service.ts | 205 +++++++++++++-------------
```

…au lieu de :

```
 public.service.ts | 5 +++--
```

Conséquences :
- ❌ diff **illisible** → impossible à reviewer ;
- ❌ impossible de dire ce qui a **vraiment** changé ;
- ❌ si un levier casse quelque chose, on ne sait plus **quelle ligne** accuser ;
- ❌ la mesure « avant/après » par levier perd tout son sens.

---

## 3. Pourquoi on n'en parle « jamais » d'habitude

En dev normal, **on s'en fout** : qu'un commit reformate 200 lignes, tant pis. Ça ne devient un
problème **que** quand on veut des diffs chirurgicaux + une mesure par levier — c.-à-d. le mode
« propre » d'une refonte. **C'est un souci de méthode de refonte, pas de qualité de code.**

---

## 4. D'où vient le reformatage automatique (les mécanismes)

| Mécanisme | Où | Effet |
|---|---|---|
| `editor.formatOnSave: true` | settings VS Code (user ou workspace) | reformate le fichier entier à chaque `Cmd+S` |
| `editor.codeActionsOnSave` → `source.fixAll.eslint` | settings VS Code | ESLint applique ses fixes à la sauvegarde ; avec `eslint-plugin-prettier`, ça **reformate** aussi |
| `source.organizeImports` | settings VS Code | réordonne/supprime les imports à la sauvegarde |
| Bloc par langage `"[typescript]": { "editor.formatOnSave": true }` | settings VS Code | réactive le format-on-save juste pour un langage |
| `Format Document` (Shift+Alt+F) | action manuelle | reformate tout le fichier |
| `npm run format` | script repo (`prettier --write "src/**/*.ts"`) | reformate **tout** `src/` d'un coup |

> ⚠️ Piège insidieux : le repo `attendee-ems-back` a `eslint-plugin-prettier`. Donc même sans
> `formatOnSave`, un `source.fixAll.eslint` à la sauvegarde peut reformater via ESLint.

---

## 5. Comment vérifier (macOS / VS Code)

**a) Lire les réglages VS Code :**

```bash
# Chercher tout mécanisme de formatage à la sauvegarde
grep -nE 'formatOnSave|codeActionsOnSave|source\.fixAll|"\[(type|java)script\]"|source\.organizeImports' \
  "$HOME/Library/Application Support/Code/User/settings.json"
```

- Si `editor.formatOnSave` est **absent** → défaut VS Code = `false` (rien ne reformate).
- Si `source.fixAll.eslint` / `organizeImports` sont **absents** → pas de reformatage déguisé.

**b) Voir la config Prettier du repo :**

```bash
ls -a | grep -iE 'prettier|editorconfig'   # .prettierrc, .editorconfig ?
grep -iE 'prettier|format' package.json     # script "format", "prettier" key ?
```

**c) Test empirique (preuve à 100 %) :** ouvrir un `src/**/*.ts`, faire `Cmd+S`, puis :

```bash
git status   # doit être VIDE. Si le fichier apparaît modifié → format-on-save est ON.
```

---

## 6. Pourquoi, dans NOTRE cas, ce n'est pas un souci

Vérifié le 2026-07-06 sur `attendee-ems-back` :

- **Aucune config Prettier committée** dans le repo (`.prettierrc`, `.editorconfig`, `.vscode/settings.json`
  absents) → Prettier ne tourne que si on l'appelle **à la main**.
- **`editor.formatOnSave` absent** des settings user → défaut `false`, donc `Cmd+S` ne reformate pas.
- **Pas de `source.fixAll.eslint` ni `organizeImports`** à la sauvegarde → pas de reformatage
  déguisé via ESLint.
- **Contexte solo** (travail sur `dev` avant MR vers `staging`) → pas de config d'un autre dev
  qui viendrait reformater différemment.

**Conclusion : l'étape 1 était déjà satisfaite, rien à modifier.** Le seul risque résiduel est
**humain** : lancer `npm run format` ou `Format Document` par réflexe.

---

## 7. La règle à retenir

Pendant une refonte « 1 levier = 1 commit = 1 mesure » :

- ✅ garder le **format-on-save OFF** (ou vérifier qu'il l'est) ;
- ❌ ne **pas** lancer `npm run format` ;
- ❌ ne **pas** faire `Format Document` (Shift+Alt+F) sur les fichiers qu'on touche ;
- ✅ si un reformatage global est un jour souhaité → le faire dans **un seul commit `style: prettier`
  isolé**, jamais mêlé à un commit de logique.

→ Résultat : des diffs chirurgicaux, reviewables, et une mesure fiable par levier.
