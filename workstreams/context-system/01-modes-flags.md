# 01 — Modes d'usage & flags

## Objectif

Couvrir les cas d'usage réels sans sur-construire. Un dev (ou l'IA) doit pouvoir choisir **combien** de contexte il charge et **avec quel niveau de confiance**.

---

## Les cas d'usage réels

| # | Situation | Ce que le dev veut |
|---|---|---|
| A | « Je bosse sur l'auth API » | Contexte **partiel** ciblé sur un scope |
| B | « Je découvre le projet » | Contexte **full** (tout) |
| C | « Je veux juste enrichir/MAJ le contexte » | **Ne pas** charger comme source de vérité, juste éditer les fichiers |
| D | « Je ne veux pas du contexte » | Ne rien faire — pas de flag, on ne lance pas le script |
| E | « Je veux du contexte mais SEULEMENT ce qui est à jour » | **Fresh-only** — exclure les fichiers stale |
| F | « Je veux tout, mais savoir ce qui est douteux » | Bundle **annoté** avec la fraîcheur de chaque fichier |

> Cas C et D ne sont **pas** des flags : ce sont des comportements. C = on édite les `.md` à la main. D = on ne lance rien. Pas besoin de coder pour ça.

---

## Jeu de flags proposé (minimal)

Sur `get-context.js`, en plus de l'existant (`--hub`, `--local`, `--format`) :

| Flag | Cas | Effet |
|---|---|---|
| `<scope>` (positionnel) | A | `api/auth`, `api/*`, `*` |
| `--format bundle\|list` | A,B | déjà là |
| `--fresh-only` | E | exclut les fichiers dont `last-verified` > seuil critique |
| `--max-age <jours>` | E | seuil custom (défaut 120) |
| `--annotate` | F | ajoute un en-tête de fraîcheur sur chaque fichier du bundle |

**`--annotate` est le flag le plus important.** Il rend la fraîcheur **visible par l'IA** dans le bundle lui-même :

```
===== FILE: hub > constraints/security.md  [⚠️ STALE — last-verified: 2026-02-17, 125j] =====
(contenu)
```

→ L'IA sait qu'elle doit traiter ce fichier avec prudence. C'est ce qui résout le problème « comment être sûr que l'info est à jour » côté consommation.

---

## Ce qu'on NE fait pas

- Pas de flag « source de vérité oui/non » → c'est une **convention humaine**, pas un flag.
- Pas de profils sauvegardés / config files → YAGNI tant qu'on n'en a pas besoin.
- Pas de mode interactif → la ligne de commande suffit.

---

## Décision à prendre

- [ ] Valider ce jeu de 3 flags (`--fresh-only`, `--max-age`, `--annotate`).
- [ ] Implémenter `--annotate` en premier (plus haute valeur).
