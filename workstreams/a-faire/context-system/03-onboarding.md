# 03 — Onboarding & adoption

## Le vrai test

Un système de contexte ne vaut **rien** si les autres devs ne s'en servent pas. Le succès se mesure à une chose : **un autre dev l'utilise spontanément sans qu'on lui rappelle.**

Trois leviers d'adoption : **friction zéro**, **doc 1 page**, **valeur immédiate**.

---

## Levier 1 — Friction zéro

Déjà en place :
- Raccourcis npm identiques sur tous les repos (`npm run context`, `npm run context:check`).
- Une seule structure de dossiers partout (`architecture/ constraints/ decisions/ playbooks/`).

À faire :
- Que la **première commande** qu'un dev tape donne un résultat utile sans config.
  ```bash
  npm run context:check     # marche depuis n'importe quel repo, zéro argument
  ```

---

## Levier 2 — La doc 1 page (le livrable clé)

Un seul fichier, à la racine du hub : `attendee-context-hub/README.md`. Pas un pavé. Structure cible :

```
# Context Hub — comment s'en servir (5 min)

## C'est quoi ?
La doc partagée du projet, organisée pour être chargée vite (humain ou IA).

## Les 3 commandes que tu utiliseras
- npm run context "api/*"        → charger le contexte d'un domaine
- npm run context:check          → voir ce qui est à jour ou périmé
- (éditer un .md + bump last-verified) → enrichir le contexte

## La seule règle
Quand tu touches un domaine, tu mets à jour son contexte (last-verified = aujourd'hui).

## Où va quoi ?
- transversal (≥ 2 surfaces) → hub
- spécifique à un repo → <repo>/context/
```

→ Si un dev lit **uniquement** cette page, il doit pouvoir utiliser le système.

---

## Levier 3 — Valeur immédiate (le « aha »)

Le dev doit gagner quelque chose **dès la 1ère utilisation**, sinon il n'y revient pas.

Argument qui marche : **« tu colles ça dans Copilot/ChatGPT et il connaît le projet »**.
```bash
npm run context "api/auth" -- --format bundle | pbcopy
# → collé dans l'IA = elle connaît l'auth du projet, les ADR, les contraintes
```

C'est le pitch d'adoption. Pas « documente bien », mais « gagne du temps avec l'IA ».

---

## Plan d'adoption

- [ ] Écrire la doc 1 page (`hub/README.md`) — la rendre la **première** chose qu'on voit.
- [ ] Faire une démo de 5 min à l'équipe : le pitch « IA qui connaît le projet ».
- [ ] Ajouter le pitch + les 3 commandes dans le `.github/copilot-instructions.md` de chaque repo (pour que l'IA elle-même rappelle d'utiliser le contexte).
- [ ] Observer : est-ce qu'un autre dev lance `npm run context` sans qu'on lui dise ? = métrique de succès.

---

## Anti-objectifs

- Pas de formation longue, pas de wiki, pas de process lourd.
- Si ça demande plus de 5 min à comprendre, c'est raté → simplifier.
