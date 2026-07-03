# Comprendre Passport, JwtAuthGuard et JwtStrategy dans NestJS

> Référence personnelle pour comprendre le flow d'authentification JWT.
> Backend : `attendee-ems-back/`

---

## 1. C'est quoi Passport ?

**Passport** est une librairie Node.js qui standardise l'authentification.
Elle ne fait **pas** d'authentification toute seule. Elle fournit un **squelette** dans lequel tu branches des **stratégies** (jwt, google, facebook, local, etc.).

Une stratégie répond à 1 question : **"comment vérifier que ce user est qui il prétend être ?"**

| Stratégie | Comment ? |
|-----------|-----------|
| `passport-local` | Vérifier email + password |
| `passport-jwt` | Vérifier un JWT signé |
| `passport-google-oauth20` | Rediriger vers Google OAuth |

Tu peux **enregistrer plusieurs stratégies** par nom : `'jwt'`, `'jwt-refresh'`, `'google'`, etc.

---

## 2. NestJS rajoute une couche par-dessus Passport

NestJS encapsule Passport dans `@nestjs/passport` avec **2 classes** importantes :

| Classe NestJS | Rôle | Hérite de |
|---------------|------|-----------|
| `PassportStrategy(Strategy)` | Définir **comment** valider | `passport-jwt`'s `Strategy` |
| `AuthGuard('jwt')` | Le **guard** qui déclenche la stratégie | `CanActivate` de Nest |

Et tu te retrouves avec :

```
JwtStrategy        ← extends PassportStrategy(Strategy)  ← extends passport-jwt Strategy
JwtAuthGuard       ← extends AuthGuard('jwt')            ← extends CanActivate
```

---

## 3. `AuthGuard('jwt')` — C'est quoi exactement ?

`AuthGuard` est une **fonction qui retourne une classe** (un pattern courant en TS appelé "mixin").

```ts
const SomeGuard = AuthGuard('jwt');   // ← retourne une CLASSE
class MyGuard extends SomeGuard {}    // ← tu peux étendre cette classe
```

Le **paramètre `'jwt'`** est crucial : c'est **le nom de la stratégie** à utiliser.

Quand tu écris `AuthGuard('jwt')`, ça veut dire :
> "Crée-moi un guard qui, quand il s'active, va chercher la stratégie nommée `'jwt'` enregistrée dans Passport et l'exécute."

- `AuthGuard('google')` → déclenche la stratégie Google
- `AuthGuard('jwt-refresh')` → déclenche une autre stratégie JWT (pour les refresh tokens)

### Comment la stratégie `'jwt'` est-elle "nommée" ?

Dans `JwtStrategy` :

```ts
export class JwtStrategy extends PassportStrategy(Strategy) {
//                                              ^^^^^^^^
//                            'jwt' par défaut (nom de la stratégie passport-jwt)
```

Par défaut, la classe `Strategy` de `passport-jwt` s'enregistre sous le nom `'jwt'`. Tu peux changer ça avec `PassportStrategy(Strategy, 'mon-nom')`.

---

## 4. `super.canActivate(context)` — Que fait-il ?

`canActivate()` est la méthode standard d'un **Guard NestJS**. Elle retourne `true` (laisser passer) ou `false`/throw (bloquer).

Quand tu fais `super.canActivate(context)`, tu appelles **la méthode `canActivate` de la classe parent**, c'est-à-dire celle générée par `AuthGuard('jwt')`.

Cette méthode parent fait, sous le capot :

```
1. Récupère la stratégie 'jwt' enregistrée dans Passport
2. Appelle ExtractJwt.fromAuthHeaderAsBearerToken()(request)
   → lit le header "Authorization: Bearer xxx"
3. Vérifie la signature du JWT avec secretOrKey
4. Si la signature est invalide ou le token expiré → throw UnauthorizedException
5. Si OK → décode le payload (le contenu du JWT)
6. Appelle JwtStrategy.validate(payload)   ← TA méthode
7. Le retour de validate() devient request.user
8. Retourne true → la requête continue
```

**TL;DR** : `super.canActivate()` = "fais le travail standard d'authentification JWT".

---

## 5. Relation entre `JwtAuthGuard` et `JwtStrategy`

C'est la partie la plus importante. Imagine ça comme **chef d'équipe + ouvrier spécialisé** :

```
         HTTP Request
              │
              ▼
    ┌─────────────────────┐
    │  JwtAuthGuard       │  ← LE CHEF
    │  (canActivate)      │
    └─────────┬───────────┘
              │ Si pas @Public, appelle super.canActivate()
              ▼
    ┌─────────────────────┐
    │ AuthGuard('jwt')    │  ← LE CADRE NESTJS/PASSPORT
    │ (machinerie         │
    │  Passport)          │
    └─────────┬───────────┘
              │ Cherche la stratégie 'jwt'
              │ Extrait le token
              │ Vérifie la signature
              │ Décode le payload
              ▼
    ┌─────────────────────┐
    │  JwtStrategy        │  ← L'OUVRIER (TA logique)
    │  .validate(payload) │
    │  → check user.is_active
    │  → retourne req.user│
    └─────────┬───────────┘
              │ Le retour devient req.user
              ▼
        Controller
```

### Qui fait quoi, en clair :

| Fichier | Rôle |
|---------|------|
| [src/common/guards/jwt-auth.guard.ts](../../../../../../attendee-ems-back/src/common/guards/jwt-auth.guard.ts) | **Décide quand activer** la machinerie JWT (skip si `@Public`) |
| [src/auth/jwt.strategy.ts](../../../../../../attendee-ems-back/src/auth/jwt.strategy.ts) | **Décide comment** extraire/vérifier le JWT (config dans `super({...})`) ET **ce qui devient `req.user`** (méthode `validate`) |
| Lib `@nestjs/passport` + `passport-jwt` | Fait le **travail technique** (lire header, vérifier signature, etc.) |

### Le `super({...})` dans `JwtStrategy`

```ts
super({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  ignoreExpiration: false,
  secretOrKey: configService.jwtAccessSecret,
});
```

Tu **configures** la stratégie au moment de la créer :
- Où trouver le token ? → `Authorization: Bearer xxx`
- Faut-il accepter les tokens expirés ? → Non
- Avec quelle clé secrète vérifier la signature ? → celle du config

C'est cette config qui sera utilisée par la lib `passport-jwt` quand `super.canActivate()` est appelée plus tard.

### La méthode `validate(payload)`

```ts
async validate(payload: any): Promise<JwtPayload> {
  const user = await this.prisma.user.findUnique({...});
  if (!user || !user.is_active) throw new UnauthorizedException(...);
  return { sub, id, mode, currentOrgId, iat, exp };
}
```

Elle est appelée **après** que la signature JWT a été vérifiée par Passport.
- `payload` = le contenu décodé du JWT (ce qui était dans `jwt.sign(...)` côté login)
- Tu peux faire des checks supplémentaires (ici : "l'user existe-t-il toujours en DB et est-il actif ?")
- **Ce que tu retournes devient `req.user`**.

C'est pour ça que dans tes controllers tu fais `@Req() req` puis `req.user.sub`, `req.user.mode`, etc.

---

## 6. Le cycle complet (résumé visuel)

```
User → POST /attendees avec header "Authorization: Bearer eyJhbGc..."
         │
         ▼
   [JwtAuthGuard.canActivate]
         │
         ├── @Public ? → Non
         │
         └── super.canActivate(context)
                 │
                 ▼
            [Machinerie Passport, transparente]
                 │
                 ├── Lit "Bearer eyJ..."
                 ├── Vérifie signature avec secret
                 ├── Vérifie pas expiré
                 ├── Décode → payload = { sub: 'user-123', mode: 'tenant', ... }
                 │
                 └── JwtStrategy.validate(payload)
                         │
                         ├── Cherche user en DB
                         ├── Vérifie is_active
                         └── return { sub, id, mode, currentOrgId, ... }
                                 │
                                 ▼
                           req.user = ce retour

   ◄────────── true ────────────────
         │
         ▼
   [RequirePermissionGuard]   ← le suivant dans la chaîne
         │
         ▼
   [Controller.findAll(req.user)]
```

---

## 7. Analogie pour ancrer

Imagine un **bâtiment sécurisé** :

- **`JwtAuthGuard`** = le **portier** à l'entrée. Il décide : "cette personne a-t-elle besoin d'un badge ? (`@Public` = non, sinon oui)". S'il faut un badge, il dit "passez au scanner".

- **`AuthGuard('jwt')` parent** = le **scanner de badges**. Il scanne, vérifie l'authenticité.

- **`JwtStrategy.validate()`** = le **bureau RH** derrière. "Ok le badge est authentique. Mais cette personne est-elle encore employée ? (`is_active`). Oui ? Voici son profil détaillé que tu peux donner à tout le monde dans le bâtiment (`req.user`)".

- **`RequirePermissionGuard`** (vient ensuite) = le **vigile à l'étage** qui regarde le profil et dit "oui tu peux entrer dans cette salle".

---

## 8. Points clés à retenir

1. **Passport** = framework d'auth, neutre. Tu lui branches des stratégies.
2. **NestJS** wrappe Passport avec deux classes : `PassportStrategy` (la stratégie) et `AuthGuard` (le guard).
3. **`AuthGuard('jwt')`** = "guard qui exécute la stratégie nommée `'jwt'`".
4. **`super.canActivate()`** dans le guard = délègue au mécanisme Passport (extrait token, vérifie signature, appelle `validate`).
5. **`JwtStrategy.validate(payload)`** = TA logique métier post-vérification. Le retour devient `req.user`.
6. **`JwtAuthGuard`** (custom) ne fait qu'une chose en plus : skip si `@Public`. Tout le reste est délégué.
7. Le **JWT signé** est vérifié par `passport-jwt` (lib externe), pas par toi. Tu ne fais que :
   - configurer (clé, expiration, où lire le token)
   - enrichir/refuser après vérif (`validate`)

---

## 9. Lecture des fichiers concernés

- [src/common/guards/jwt-auth.guard.ts](../../../../../../attendee-ems-back/src/common/guards/jwt-auth.guard.ts) — le wrapper
- [src/auth/jwt.strategy.ts](../../../../../../attendee-ems-back/src/auth/jwt.strategy.ts) — la stratégie (config + validate)
- `@nestjs/passport` — la couche d'abstraction NestJS
- `passport-jwt` — la lib qui fait le vrai boulot de vérification JWT
