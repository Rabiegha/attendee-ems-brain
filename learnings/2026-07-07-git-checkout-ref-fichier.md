# Git : `checkout <ref> -- fichier` & `reset --soft` — réécrire proprement

> **Date :** 2026-07-07 · **Contexte :** rejeu propre des leviers scaling (chantier A LFD 2026).
> **Deux outils :** `checkout <ref> -- fichier` (récupérer un fichier d'une autre réf, sans merge)
> et `reset --soft` (rembobiner l'historique en gardant le travail). Piège clé : `--amend` après
> un push crée un doublon.

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

---

# `git reset --soft <ref>` — rembobiner l'historique, garder le travail

> **En une phrase :** `reset --soft` **déplace le pointeur de la branche** sur `<ref>` (annule les
> commits d'après) **mais conserve tous leurs changements indexés**, prêts à recommitter.

## Ce que ça fait

```bash
git reset --soft 4d58298    # rembobine la branche sur ce commit, garde tout le travail
```

- **L'historique** entre `<ref>` et la position actuelle est effacé de la branche.
- **Les fichiers** ne bougent pas : leurs changements restent **stagés** (prêts à `git commit`).
- Ça peut annuler **plusieurs** commits d'un coup (tout ce qui est entre `<ref>` et `HEAD`), pas
  juste le dernier. Pour le dernier seulement : `git reset --soft HEAD~1`.

## Les 3 modes de reset

| Mode | Pointeur | Changements gardés ? | Où ils atterrissent |
|---|---|---|---|
| `--soft` | rembobiné | ✅ oui | **indexés** (prêts à committer) |
| `--mixed` (défaut) | rembobiné | ✅ oui | working tree (à re-`git add`) |
| `--hard` | rembobiné | ❌ **non** | **perdus** ⚠️ (destructeur) |

## ⚠️ Le piège qui nous a menés là : `--amend` APRÈS un push

Un commit est **immuable**. `git commit --amend` ne modifie pas le commit : il en **crée un
nouveau** (nouveau SHA) et jette l'ancien **en local**.

- Si le commit **n'a pas encore été poussé** → sans souci, personne d'autre ne l'a vu.
- Si le commit **a déjà été poussé** → GitHub garde l'**ancien** (A), toi tu as le **nouveau**
  (A'). Au `pull`/`push` suivant, git **réconcilie par un merge** → tu te retrouves avec
  **A + A' + un commit de merge** : un **doublon** pour un seul vrai changement.

## La réparation (ce qu'on a fait)

```bash
git reset --soft <base>          # rembobine les commits moches, garde les fichiers
git rm -f <fichier-en-trop>      # (si besoin) retire un fichier qui ne doit pas y être
git add -A                       # re-stage tout (y compris les correctifs)
git commit -m "…"                # UN seul commit propre
git push --force-with-lease      # réécrit l'historique distant (garde-fou : refuse si
                                 #   quelqu'un a poussé entre-temps)
```

## Règles à retenir

- **Ne jamais `--amend` (ni `rebase`) un commit déjà poussé** sur une branche **partagée**.
- Sur une branche **perso non mergée**, réécrire est OK → nettoyer avec `reset --soft` +
  `push --force-with-lease` (jamais `--force` seul).
- `reset --soft` = « j'ai mal découpé mes commits, je refais l'histoire **sans perdre** mon
  travail ». `--hard` = destructeur, à éviter sauf si on veut vraiment jeter les changements.
