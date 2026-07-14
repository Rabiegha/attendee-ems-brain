# Chantier M — Appli mobile LFD2026

## But

Sortir une version mobile stable et exploitable pour l'event LFD2026.

Source backlog: `backlog/en-cours/appli-mobile-lfd2026.md`.

## Priorites

### P0 Event mandatory

1. Bug ejection mobile (refresh rotation / HTTP 499)
2. Offline fiable pour scans en session (zero perte)
3. Couverture use cases scan + correction faux succes + idempotence locale
4. Perf 20k attendees + refonte sync/pagination (a traiter ensemble)
5. Sync locale de la liste des scans session
6. Bug recherche entreprise (`attendee.company` absent de l'index)
7. Securite PrintNode key (rotation + EAS secrets)
8. OTA prod (strategie runtimeVersion definitive)
9. Faux positif impression COMPLETED cote print-client

### P1 Nice to have

1. Mot de passe oublie
2. Pagination sessions
3. Recherche email + suggestions

### P2 Post-event

1. Refonte check-in/check-out historique
2. UX connexion remember me
3. Flash donnees ancien user apres logout/login
4. Flash ecran login au demarrage

## Plan d'execution recommande

- Etape M1 (fiabilite): ejection 499 + offline scans + faux succes/idempotence locale
- Etape M2 (charge): sync/pagination robuste + perf 20k + recherche entreprise
- Etape M3 (release safety): OTA, secrets PrintNode, print-client complete check
- Etape M4 (finition): points P1 selon capacite

## Definition of done minimale event

- aucun logout force non intentionnel pendant scan en conditions reseau degradees
- aucun scan perdu offline->online
- comportement clair pour deja scanne / invalide / mauvais event-session
- perf acceptable sur events volumineux (liste + recherche)
- release mobile event produite et testee en smoke terrain
