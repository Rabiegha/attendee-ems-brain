# Rabbit Hole — Full custom domains now

Type: Rabbit Hole
Status: Watchlist
Risk level: High

## 1. Why it is tempting

- Permettre à chaque org d'avoir `events.acme.com` au lieu de `acme.attendee.app`.
- Argument commercial premium (white-label).
- Plus pro côté communication client.
- Technique apparemment simple côté Nginx / Cloud Run.

## 2. Why it is dangerous now

- **DNS automation** complexe (CNAME, A, TXT verification, propagation).
- **TLS automation** (Let's Encrypt / Cloud Certificate Manager) à gérer à grande échelle (rate limits, renewals, fallbacks).
- **Routing edge** : middleware qui résout le Host header → tenant. Coûteux à debug, source de fuites cross-tenant si raté.
- **CORS / cookies / OAuth callbacks** explosent : chaque domain custom devient une nouvelle origin.
- **WebSocket** : sticky session + CORS + Origin check à recalibrer (déjà permissif aujourd'hui, voir [../bugs/cors-origin-security-review.md](../bugs/cors-origin-security-review.md)).
- **Public namespaces** ([../backlog/public-landing-page-namespaces.md](../backlog/public-landing-page-namespaces.md)) ne sont pas encore implémentés → on ferait l'étape 3 sans avoir fait l'étape 1.
- Aucun besoin commercial concret aujourd'hui.

## 3. Scope creep risk

Énorme :
- Tableau de bord d'ajout / suppression de domain.
- Worker de vérification DNS périodique.
- Worker de renewal TLS.
- Pages d'erreur custom par domain.
- Migration des emails outbound (DKIM / SPF par tenant ?).
- Compatibilité multi-domain par org (production vs staging).
- Logs et tracing par domain.

## 4. Current decision

- **Pas maintenant.**
- Démarrer d'abord par les **Public Namespaces** (sous-domaines `*.attendee.app`) — voir [../backlog/public-landing-page-namespaces.md](../backlog/public-landing-page-namespaces.md).
- Custom domains restent un sujet **futur** subordonné à un besoin client identifié.

## 5. When to revisit

- Quand un client paye explicitement pour un custom domain.
- Quand les Public Namespaces seront en place et stables.
- Quand l'équipe ops est prête à gérer DNS + TLS automation.

## 6. Related docs

- [../backlog/public-landing-page-namespaces.md](../backlog/public-landing-page-namespaces.md)
- [../steering/DECISIONS.md](../steering/DECISIONS.md#public-namespaces)
- [../bugs/cors-origin-security-review.md](../bugs/cors-origin-security-review.md)

## 7. Rule for Copilot

- Si une proposition contient « custom domain », « domaine personnalisé », « CNAME tenant », « Let's Encrypt auto » → **stop**, pointer vers ce document.
- Interdit : ajouter un champ `custom_domain` à `Organization`, créer un middleware de routing par Host, installer un module de cert auto.
- Autorisé : compléter la fiche [../backlog/public-landing-page-namespaces.md](../backlog/public-landing-page-namespaces.md) avec ce qu'il faudra prévoir un jour.
