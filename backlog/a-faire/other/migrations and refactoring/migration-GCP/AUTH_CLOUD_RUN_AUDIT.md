# Audit Authentification — Compatibilité Cloud Run (Serverless)

> **Date** : 2 avril 2026  
> **Projet** : attendee-ems-back (NestJS)  
> **Cible** : Google Cloud Run (stateless, scaling horizontal)  
> **Statut** : ✅ Compatible — améliorations recommandées

---

## 1. Génération des tokens

### Access Token

- **Technologie** : JWT signé via `@nestjs/jwt`
- **Secret** : `JWT_ACCESS_SECRET` (variable d'environnement)
- **Durée** : `JWT_ACCESS_TTL` (15 minutes par défaut)
- **Payload minimal** :
  ```json
  {
    "sub": "user-uuid",
    "mode": "tenant | platform",
    "currentOrgId": "org-uuid (optionnel)",
    "iat": 1234567890
  }
  ```
- **Fichier** : `src/auth/auth.service.ts` → `rotateRefreshToken()` et `generateJwtForOrg()`

### Refresh Token

- **Technologie** : JWT signé avec un **secret séparé** `JWT_REFRESH_SECRET`
- **Durée** : `JWT_REFRESH_TTL`
- **Payload** :
  ```json
  {
    "sub": "user-uuid",
    "jti": "uuid-v4-unique"
  }
  ```
- **Fichier** : `src/auth/auth.service.ts` → `issueRefreshToken()`

---

## 2. Stockage des refresh tokens

Les refresh tokens sont stockés **en base de données PostgreSQL** via Prisma, table `refresh_tokens` :

```
model RefreshToken {
  id           String    @id @default(cuid())
  userId       String    @db.Uuid
  jti          String    @unique          ← identifiant unique du token
  tokenHash    String                     ← SHA-256 du token brut (jamais le token en clair)
  userAgent    String?
  ip           String?
  createdAt    DateTime  @default(now())
  expiresAt    DateTime
  revokedAt    DateTime?                  ← null = actif, date = révoqué
  replacedById String?
}
```

**Points positifs** :
- Le token brut n'est jamais stocké, seul le hash SHA-256 est persisté
- `jti` unique permet l'identification et la révocation ciblée
- `revokedAt` permet un soft-delete (traçabilité)

---

## 3. Vérification du refresh token (`POST /auth/refresh`)

Flux dans `rotateRefreshToken()` :

```
1. jwtService.verify(oldToken, { secret: JWT_REFRESH_SECRET })
   → Vérifie signature + expiration cryptographique

2. Calcul SHA-256 du token brut
   → Recherche en DB : jti + tokenHash + revokedAt: null

3. Vérification expiration DB (storedToken.expiresAt < now)

4. Chargement user complet (+ vérification is_active)

5. Révocation de l'ancien token (revokedAt = now)

6. Génération nouveau access token (JWT minimal)

7. Génération nouveau refresh token (rotation complète)
```

C'est un système de **refresh token rotation** : chaque utilisation invalide l'ancien token et en émet un nouveau.

**Transport du refresh token** :
- **Web** : cookie `HttpOnly`, `Secure`, `SameSite` (configurable)
- **Mobile** : body de la requête (détecté via header `x-client-type: mobile`)

---

## 4. Dépendances au stockage mémoire côté serveur

| Composant | Stockage | Stateless ? |
|---|---|---|
| Access token | JWT auto-contenu (vérifié par signature) | ✅ |
| Refresh token | PostgreSQL (table `refresh_tokens`) | ✅ |
| Sessions utilisateur | Aucune (pas de session store) | ✅ |
| Permissions | Chargées depuis la DB à chaque requête | ✅ |
| Configuration | Variables d'environnement | ✅ |
| Cache | Aucun cache in-memory | ✅ |

**Aucune dépendance à la mémoire locale du serveur.**

---

## 5. Verdict : Stateless ou non ?

### ✅ Le système est 100% stateless

- Aucun session store en mémoire
- Aucun cache in-memory (pas de Redis, pas de `Map` locale)
- Aucune variable globale mutable
- Tous les états persistants sont en DB
- Le `JwtStrategy` valide chaque requête via DB + signature JWT, sans état local

**Compatible avec le scaling horizontal et Cloud Run.**

---

## 6. Risques identifiés pour Cloud Run

### ⚠️ Race condition sur le refresh token rotation (Sévérité : MOYENNE)

Avec plusieurs instances, si le frontend envoie 2 refresh simultanés (ex: 2 onglets, retry réseau) :
1. Instance A reçoit le refresh, révoque l'ancien token, émet un nouveau
2. Instance B reçoit le même refresh (déjà révoqué) → **401 → déconnexion inattendue**

**Impact** : L'utilisateur est déconnecté de façon aléatoire sous charge.

### ⚠️ Connection pool DB (Sévérité : MOYENNE)

Chaque instance Cloud Run ouvre son propre pool Prisma.  
Si 10 instances × 10 connections = 100 connections DB simultanées.

### ⚠️ Cold starts (Sévérité : FAIBLE)

NestJS prend ~2-5s au démarrage. Les premières requêtes après un scale-up seront lentes.

### ⚠️ Requêtes DB par requête authentifiée (Sévérité : FAIBLE)

`JwtStrategy.validate()` fait un `SELECT` pour chaque requête authentifiée.  
Avec du scaling horizontal, la charge DB augmente linéairement.

### ✅ Cookies cross-instance (Pas de problème)

Les cookies sont gérés côté client, aucun problème multi-instance.

---

## 7. Améliorations recommandées

### Sécurité

#### 7.1 Détection de réutilisation de refresh token (PRIORITÉ HAUTE)

Si un token déjà révoqué est réutilisé, c'est un indicateur de vol. Il faut révoquer toute la session :

```typescript
// Dans rotateRefreshToken(), quand storedToken n'est pas trouvé :
if (!storedToken) {
  // Token volé → révoquer TOUS les tokens du user
  await this.revokeAllUserSessions(userId);
  throw new UnauthorizedException('Token reuse detected — all sessions revoked');
}
```

#### 7.2 Supprimer les console.log sensibles en production

Les logs actuels leakent des informations :
```
[validateUser] Password valid: true/false  ← à supprimer
[login] User after validation: user@email.com  ← à supprimer
```

Recommandation : utiliser un logger conditionnel (`if (NODE_ENV !== 'production')`).

#### 7.3 Rate limiting sur les endpoints auth

Aucune protection brute-force visible sur `/auth/login` et `/auth/refresh`.  
Ajouter `@nestjs/throttler` ou un rate limiter Cloud Run.

### Scalabilité (Cloud Run)

#### 7.4 Grace period pour le refresh token rotation (PRIORITÉ HAUTE)

Ajouter une fenêtre de tolérance (~10s) après révocation pour gérer les requêtes concurrentes :

```typescript
// Au lieu de : revokedAt: null
const storedToken = await this.prisma.refreshToken.findFirst({
  where: {
    jti,
    tokenHash: oldTokenHash,
    OR: [
      { revokedAt: null },
      { revokedAt: { gte: new Date(Date.now() - 10_000) } } // grace 10s
    ],
  },
});
```

#### 7.5 PgBouncer / Prisma Accelerate devant PostgreSQL

Mutualiser les connexions DB entre instances Cloud Run pour éviter l'explosion du nombre de connexions.

#### 7.6 Cache Redis optionnel pour JwtStrategy.validate()

Réduire les `SELECT user WHERE id = ?` à chaque requête authentifiée :
- Clé : `user:active:{userId}`
- TTL : 60s
- Invalidation : sur désactivation de compte

Sans cache, ça fonctionne mais chaque requête API = 1 query DB minimum.

#### 7.7 Configurer `min_instances: 1` sur Cloud Run

Éviter les cold starts sur les requêtes critiques (auth, refresh).

---

## Résumé

| Critère | Statut |
|---|---|
| Architecture stateless | ✅ |
| Tokens en DB (pas en mémoire) | ✅ |
| Secrets en variables d'env | ✅ |
| Refresh token rotation | ✅ |
| Hash SHA-256 (pas de token brut en DB) | ✅ |
| Cookie HttpOnly + Secure | ✅ |
| Support mobile (body) + web (cookie) | ✅ |
| Race condition multi-instance | ⚠️ À corriger (grace period) |
| Détection de token reuse | ⚠️ À implémenter |
| Rate limiting | ⚠️ Absent |
| Connection pooling | ⚠️ À configurer (PgBouncer) |

**Le système est prêt pour Cloud Run.** Le seul correctif bloquant est la grace period sur le refresh token rotation (point 7.4).
