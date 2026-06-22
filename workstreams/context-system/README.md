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
| **Fraîcheur visible DANS le bundle IA** | ❌ pas encore |
| **Doc onboarding 1 page** | ❌ pas encore |
| **Modes/flags d'usage** | ⚠️ partiels (bundle/list seulement) |

---

## Parties

| Partie | Sujet | Status |
|---|---|---|
| [01](01-modes-flags.md) | Modes d'usage & flags (full / partiel / fresh-only / feed-only) | 📋 À faire |
| [02](02-freshness.md) | Garantir la fraîcheur — boucle de mise à jour + fraîcheur dans le bundle | 📋 À faire |
| [03](03-onboarding.md) | Onboarding & adoption — doc 1 page + rituel d'équipe | 📋 À faire |

---

## Prochaine action

➡️ Décider du jeu de flags minimal (partie 01) avant de coder quoi que ce soit, pour ne pas sur-construire.

---

## Hors-scope

- Réécrire `get-context.js` from scratch (on étend, on ne refait pas).
- Automatisation CI/CD du check de fraîcheur (plus tard, voir backlog).
- Remplir le contenu métier des `context/` vides (c'est un chantier séparé, par repo).
