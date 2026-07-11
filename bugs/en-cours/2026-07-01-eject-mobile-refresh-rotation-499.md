# Éjection mobile — rotation refresh token perdue (HTTP 499)

> **Date incident** : 2026-07-01 ~19:18 (Paris) · **App** : mobile (iOS, `Attendee/27 CFNetwork/1496.0.7`)
> **User impacté** : `acbru@choyou.fr` · **Contexte** : event « Dîner débat CYBERARK »
> **Status** : Confirmé, non corrigé — investigation du 2026-07-02 (logs nginx prod + table `refresh_tokens`)

## Symptôme

En ouvrant l'app mobile pendant l'event, l'utilisatrice est éjectée → retour à la page login.
Reconnexion sur ce device seulement le lendemain matin (02/07 10:41 Paris, login frais).

## Timeline (logs nginx `ems-nginx` + DB prod, IP device `193.3.21.12`)

| Heure Paris | Événement |
|---|---|
| 18:59:22 | `POST /auth/refresh` → **200** — rotation OK (token `07f81341` émis) |
| 19:13:34 | `POST /auth/refresh` → **499** (`rt=0.004`) — **le client coupe la connexion avant la réponse**. Côté serveur la rotation a quand même eu lieu : `07f81341` révoqué à 17:13:34.542 UTC, nouveau token `4ee3e9bb` créé 12 ms après — mais la réponse n'atteint jamais le device |
| 19:18:36 | Retry `POST /auth/refresh` avec **l'ancien token déjà révoqué** → **401** `Refresh token not found or revoked` → éjection vers login |

## Cause racine

**Désynchronisation de rotation de refresh token** (pattern connu : rotation stricte sans grace period).

1. Le serveur révoque l'ancien token **avant** que le client ait reçu le nouveau.
2. Le HTTP 499 = l'app a abandonné la requête (réseau mobile instable du lieu de l'event, ou app suspendue par iOS en arrière-plan).
3. Au retry, l'app présente un token mort → 401 → logout forcé.

Code : `attendee-ems-back/src/auth/auth.service.ts` → `rotateRefreshToken()` rejette
immédiatement tout token avec `revokedAt != null`, sans tolérance.

## Pas lié à l'incident 502 du 25/06

Vérifié explicitement (soupçon initial) — **aucun lien** :
- `ems-api` up sans interruption depuis le 25/06 00:17 UTC (aucun restart le 01/07).
- Secrets JWT inchangés : le token mobile du même device créé le **16/06** a été roté avec succès le 01/07 18:59 ; l'autre device (`Attendee/8`) a une chaîne de rotation intacte du 25/06 au 01/07.
- Aucune purge de `refresh_tokens` (historique complet depuis avril présent).
- L'incident 502 (collision nom de projet Docker, fix `5f72daa`) a duré quelques minutes dans la nuit du 25/06, sans impact sur les sessions.

## Pistes de correction

1. **Backend — grace period sur la rotation** : accepter un token révoqué depuis < 60 s en
   renvoyant le token qui l'a remplacé. La colonne `replacedById` existe déjà dans
   `refresh_tokens` mais n'est **jamais renseignée** (vide sur toutes les lignes) — la
   peupler dans `rotateRefreshToken()` puis s'en servir pour le fallback.
2. **Mobile — retry sans jeter le token** : sur échec *réseau* du refresh (timeout,
   coupure, 499 côté serveur), retenter avec backoff **sans** invalider le refresh token
   local tant qu'un vrai 401 HTTP n'a pas été reçu.

## Risque

Reproduction probable en conditions événementielles (réseau saturé/instable, app
backgroundée entre deux scans) — contexte typique LFD2026.
