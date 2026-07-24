# Workstream — Fiabilisation des refresh tokens

## Pilote

```text
App principale : attendee-ems-back
Apps concernées : attendee-ems-front, attendee-ems-mobile, attendee-ems-print-client
Branche         : fix/auth-refresh-token-hardening
Statut          : En cours
Priorité        : Haute
Créé le         : 2026-07-24
```

## Objectif

Corriger les défauts identifiés sur le cycle de vie des refresh tokens sans
modifier les secrets ni les durées actuelles (`15m` pour l'access token et
`30d` pour le refresh token).

Les changements sont d'abord développés et validés sur **staging**. Aucun
portage en production avant validation des scénarios web, mobile, impression
et multi-organisation.

## État de départ

- Pas de panne générale observée : 856 refresh réussis contre 19 rejets dans
  les logs conservés.
- Secrets live distincts.
- TTL `15m / 30d` correct.
- Huit défauts techniques confirmés ou à couvrir par un test de reproduction.

## Périmètre et ordre de traitement

| Lot | Priorité | Défaut | Résultat attendu |
|---|---:|---|---|
| 1 | P0 | Rotation non atomique | Deux refresh concurrents ne déconnectent pas arbitrairement la session. |
| 1 | P0 | Logout mobile et impression | Le token envoyé par ces clients est révoqué côté serveur, puis effacé localement. |
| 1 | P0 | Changement de mot de passe | Toutes les sessions existantes de l'utilisateur sont révoquées. |
| 1 | P0 | Logout web production | Le cookie est supprimé avec le même domaine et le même chemin que lors de sa création. |
| 2 | P1 | Multi-organisation | Le refresh conserve l'organisation active au lieu de reprendre la première. |
| 2 | P1 | Refresh token exposé au navigateur | Le navigateur reçoit le token uniquement dans le cookie HttpOnly, jamais dans le JSON. |
| 2 | P1 | Erreur DB transformée en 401 | Une panne temporaire remonte en erreur serveur réessayable ; `401` reste réservé à une session invalide. |
| 3 | P2 | Tokens expirés non nettoyés | Une tâche bornée et observable supprime régulièrement les lignes expirées. |

## Méthode

Pour chaque défaut :

1. Écrire un test qui reproduit le comportement actuel.
2. Corriger la cause la plus locale possible.
3. Vérifier qu'aucun secret ou refresh token n'est journalisé.
4. Valider le scénario sur staging avec le client concerné.
5. Consigner ici le résultat et la preuve de validation.

## Avancement

- [x] Branche dédiée créée dans `attendee-ems-back` et `attendee-ems-brain`.
- [x] Logout web : cause comprise et correctif local préparé.
- [x] Logout web : test unitaire avec le domaine production `.attendee.fr`.
- [x] Détour opérationnel du 24/07 : collision DNS Docker entre Attendee et
      Public-Form-Logger corrigée en production, sans redémarrage de l'API.
- [ ] Lot 1 terminé et validé sur staging.
- [ ] Lot 2 terminé et validé sur staging.
- [ ] Nettoyage des tokens expirés validé sur staging.
- [ ] Revue sécurité et plan de déploiement production.

## Détour opérationnel réalisé pendant ce workstream

Le diagnostic de lenteur de `staging.attendee.fr`, effectué pendant le présent
workstream, a révélé un incident réseau distinct en production.

Les stacks Attendee et Public-Form-Logger partageaient `ems-network` avec les
alias Compose génériques `api` et `postgres`. Le DNS Docker pouvait ainsi
résoudre `api` vers Public-Form-Logger (`172.18.0.3`) au lieu d'Attendee
(`172.18.0.5`), ce qui produisait des `connection refused` intermittents dans
Nginx.

Le 24/07, le correctif suivant a été appliqué :

- réseau dédié `public-form-logger-network` créé ;
- Public-Form-Logger isolé de `ems-network` ;
- Nginx raccordé aux deux réseaux nécessaires ;
- configuration Nginx testée puis rechargée à chaud ;
- Compose du backend et du logger mis à jour localement et sur le VPS ;
- 30 requêtes API consécutives et le healthcheck logger validés en `200` ;
- aucune nouvelle erreur `connection refused` après la bascule.

Ce correctif est terminé. Il est consigné ici parce qu'il a été réalisé dans
le contexte du workstream refresh token, mais il ne modifie pas son périmètre
fonctionnel.

Le reste du diagnostic de performance est reporté dans
[`staging-performance-hardening`](../staging-performance-hardening/README.md),
à traiter après la fiabilisation des refresh tokens.

## Critères de sortie staging

- Les tests unitaires, d'intégration et le build backend passent.
- Un double refresh concurrent possède un résultat déterministe et ne provoque
  pas de déconnexion injustifiée.
- Les logouts web, mobile et impression révoquent réellement le token utilisé.
- Un mot de passe modifié invalide tous les anciens refresh tokens.
- L'organisation active survit à plusieurs rotations.
- Aucune réponse web de login/refresh ne contient de `refresh_token`.
- Une indisponibilité DB simulée ne retourne pas `401`.
- Le nettoyage est borné par lots, rejouable et mesuré.

## Hors périmètre

- Rotation des secrets live.
- Modification des TTL.
- Déploiement direct en production.
- Refonte générale de l'authentification.
- Optimisation générale des performances du Dashboard et des sessions, suivie
  dans
  [`staging-performance-hardening`](../staging-performance-hardening/README.md).
