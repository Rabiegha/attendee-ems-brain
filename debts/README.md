# Debts — Dettes techniques & sécurité

Le **registre des dettes** connues de toute l'app (back, front, mobile, print-client, infra).
Vient de l'inbox `[debt]`.

> **Documenter ≠ corriger.** On ne corrige **jamais** une dette sans ticket / accord explicite.

---

## Ce qu'on met ici

- Raccourcis techniques assumés (à rembourser plus tard).
- Trous de sécurité connus (clés en clair, auth manquante…).
- Écarts entre la doc et la réalité, faux-positifs, workarounds fragiles.

Chaque dette a un identifiant (`S-xx` sécurité, `T-xx` technique), une sévérité et un « à faire ».

## Fichier

| Fichier | Contenu |
|---|---|
| [DEBTS.md](DEBTS.md) | Le registre complet, trié par domaine (sécurité, back/scaling, authz, print-client, infra). |

## Règles

- Ce fichier est la version **lisible** des notes internes de l'agent : tout ce que Copilot garde en mémoire et qui te concerne doit **aussi** atterrir dans `DEBTS.md`.
- Quand une dette est **résolue** → la retirer de `DEBTS.md` + noter dans [../learnings/README.md](../learnings/README.md) si l'apprentissage est utile.

> Pas de découpe fait / en cours / à faire ici : `DEBTS.md` est un **registre unique** où chaque ligne porte déjà son propre « à faire » et sa sévérité.
