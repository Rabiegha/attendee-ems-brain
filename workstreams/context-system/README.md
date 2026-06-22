# Workstream — Système de contexte : fiabilité & adoption

## Pilote

```
App:      transversal (context-hub + tous les repos avec context/)
Status:   Active
Priority: High
```

**Objectif :** transformer le context-hub d'un « dossier de docs » en un **système vivant et fiable** que toute l'équipe utilise sans friction.

Deux problèmes nord-étoile (tout le reste en découle) :

1. **Fraîcheur garantie** — quand on charge du contexte pour bosser (ou prompter l'IA), être **sûr** que l'info est à jour, ou au minimum **savoir** qu'elle ne l'est pas.
2. **Adoption sans friction** — un nouveau dev (ou un autre dev de l'équipe) comprend et utilise le système en < 10 min, avec une seule page de doc.

> Tout le reste (flags, scripts, formats) n'est qu'un moyen au service de ces deux buts.

---

## État actuel (2026-06-22)

| Élément | État |
|---|---|
| `context/` hub | ✅ rempli, bon contenu |
| `context/` locaux back/front | ✅ amorcés |
| `context/` mobile/print-client | ⚠️ quasi vides (.gitkeep) |
| `context/` logs/public-form-logger | ✅ structure créée |
| `last-verified` sur tous les fichiers | ✅ fait |
| Script `check-context-freshness.js` | ✅ livré (rapport + `--fix`) |
| Script `get-context.js` | ✅ existe (scope → bundle) |
| Raccourcis npm `context` / `context:check` | ✅ sur les 6 repos |
| **Fraîcheur visible DANS le bundle IA** | ✅ livré (`--annotate`, `--fresh-only`, `--max-age`) |
| **Doc onboarding 1 page** | ✅ livré (`hub/README.md`) |
| **Modes/flags d'usage** | ✅ livré (annotate / fresh-only) |
| **`copilot-instructions.md` protocole AVANT/PENDANT/APRÈS** | ✅ livré sur les 7 repos (2026-06-22) |

---

## Parties

| Partie | Sujet | Status |
|---|---|---|
| [01](01-modes-flags.md) | Modes d'usage & flags (full / partiel / fresh-only / feed-only) | ✅ `--annotate` + `--fresh-only` + `--max-age` livrés |
| [02](02-freshness.md) | Garantir la fraîcheur — boucle de mise à jour + fraîcheur dans le bundle | ✅ fraîcheur inline livrée ; reste rituel d'équipe |
| [03](03-onboarding.md) | Onboarding & adoption — doc 1 page + rituel d'équipe | ✅ doc 1 page livrée + copilot-instructions déployé sur 7 repos |

---

## Prochaine action

➡️ Démo 5 min à l'équipe (voir [03](03-onboarding.md)) + remplir `context/` mobile/print-client (par repo, en lisant le vrai code).

---

## Hors-scope

- Réécrire `get-context.js` from scratch (on étend, on ne refait pas).
- Automatisation CI/CD du check de fraîcheur (plus tard, voir backlog).
- Remplir le contenu métier des `context/` vides (c'est un chantier séparé, par repo).
