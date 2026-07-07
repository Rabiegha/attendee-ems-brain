# `git checkout <ref> -- fichier` — écraser, pas fusionner

> **Date :** 2026-07-07 · **Contexte :** rejeu propre des leviers scaling (chantier A LFD 2026).
> **En une phrase :** `git checkout <ref> -- fichier` **remplace** le fichier par sa version dans
> `<ref>` — ce n'est **pas** un merge.

---

## Ce que fait la commande

```bash
git checkout <ref> -- <chemin>
# ex : git checkout staging-archive-2026-06-25 -- docker-compose.staging.yml
```

Elle prend le fichier **tel quel** dans `<ref>` (un commit, une branche, un tag) et l'**écrase**
dans le répertoire de travail + le stage. Aucune fusion 3-way, aucune résolution de conflit :
**la version de `<ref>` gagne, point**.

---

## Pourquoi on l'a utilisée (au lieu de cherry-pick)

Le commit d'archive qui introduisait L1 était un **fourre-tout** (L1 + L2 + prisma + db-dump +
compose prod + docs). Un `cherry-pick` aurait ramené **tout le paquet**. On voulait **seulement**
les 3 fichiers de la stack staging → `git checkout <archive> -- <les 3 fichiers>`, puis un commit
propre. Résultat : diff minimal, un levier = un commit.

---

## checkout vs cherry-pick vs merge

| Commande | Effet | Fusionne ? | Quand |
|---|---|---|---|
| `git checkout <ref> -- f` | remplace `f` par la version de `<ref>` | ❌ écrase | récupérer **un/des fichiers précis** d'une autre réf |
| `git cherry-pick <commit>` | rejoue **tout le commit** | ✅ 3-way | reprendre un commit entier |
| `git merge <branche>` | fusionne les historiques | ✅ 3-way | intégrer une branche complète |

---

## ⚠️ Le piège

`git checkout <ref> -- f` **écrase les modifs locales** de `f` :
- si `f` était **committé** ailleurs → récupérable.
- si `f` avait des modifs **non committées** → **perdues** (pas dans le reflog).

**Règle : `commit` ou `stash` avant** si le fichier a des changements en cours.
Dans notre cas c'était **safe** : les fichiers étaient **nouveaux** côté destination, rien à écraser.

---

## Bon à savoir

- Voir d'abord la différence : `git diff <ref> -- f`.
- Version partielle (par morceaux) : `git checkout -p <ref> -- f`.
- Équivalent moderne (Git ≥ 2.23, plus explicite) : `git restore --source=<ref> f`.
