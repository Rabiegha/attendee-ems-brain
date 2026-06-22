# 02 — Garantir la fraîcheur

## Le problème central

Un fichier context qui ment est **pire** qu'un fichier absent : il donne une fausse confiance. Le risque #1 du système, c'est qu'on charge du contexte périmé et qu'on (ou l'IA) prenne des décisions sur une réalité dépassée.

Deux leviers : **produire** frais, et **consommer** en connaissant la fraîcheur.

---

## Levier 1 — La boucle de mise à jour (production)

La règle tient en une phrase :

> **Quand tu touches un domaine, tu touches son contexte.**

Concrètement, `last-verified` se met à jour dans 2 situations :

| Situation | Action |
|---|---|
| Tu **modifies** le contenu d'un fichier context | `last-verified` → aujourd'hui |
| Tu **lis** un fichier et confirmes qu'il est encore juste | `last-verified` → aujourd'hui |
| Tu lis mais tu n'es **pas sûr** | tu ne touches pas → la date ancienne reste un signal |

**Point d'ancrage = la PR.** Le template de PR a une case « Context / Decisions ». Le reviewer vérifie : si la PR touche l'auth et que `adr-001-auth.md` n'a pas bougé, on se pose la question.

---

## Levier 2 — La fraîcheur visible (consommation)

Deux outils, deux moments :

### Avant de bosser — `check`
```bash
npm run context:check            # rapport global
npm run context:check -- --stale-only
```
On voit en un coup d'œil les ❌ (> 120j) et ⚠️ (> 60j). C'est le **tableau de bord**.

### Au moment de charger — `--annotate` (à implémenter, voir partie 01)
Le bundle envoyé à l'IA porte la fraîcheur **inline**. L'IA lit « ce fichier a 125 jours » et adapte sa confiance. C'est la **ceinture de sécurité** : même si on oublie de vérifier avant, l'info voyage avec sa date.

---

## Le rituel minimal (pour que ça vive)

| Quand | Quoi | Coût |
|---|---|---|
| Début de session sur un domaine | `npm run context:check -- --stale-only` | 5 s |
| Tu modifies/valides un fichier | bump `last-verified` | 2 s |
| Revue de PR | vérifier la case Context | 30 s |
| 1×/mois (ou avant une grosse feature) | passer les ❌ en revue, MAJ ou marquer DEPRECATED | 15 min |

> Si un fichier est stale ET qu'on confirme qu'il est faux → on le corrige ou on le marque `DEPRECATED`. Un fichier mort se supprime, il ne traîne pas.

---

## Idée backlog (pas maintenant)

- Hook git pre-push ou CI qui **échoue** si un fichier context touché dans la PR n'a pas son `last-verified` à jour. Puissant mais intrusif → à tester sur 1 repo avant de généraliser.
