# CORS Origin — Security Review & Migration Plan

> **Statut** : Analyse / Proposition. **Aucun code modifié.**
> **Auteur** : audit interne — `feat/authz-m1-audit-service`
> **Scope** : `attendee-ems-back` — politique CORS HTTP + WebSocket
> **Référence code** : [src/main.ts](../../../attendee-ems-back/src/main.ts#L45-L77)

---

## 1. État actuel

### 1.1 Configuration HTTP (`main.ts`)

Extrait de [src/main.ts](../../../attendee-ems-back/src/main.ts#L45-L77) :

```ts
const allowedOrigins = configService.apiCorsOrigin.split(',').map(o => o.trim());
app.enableCors({
  origin: (origin, callback) => {
    // Allow requests with no origin (mobile apps, Postman, file://)
    if (!origin || origin === 'null') return callback(null, true);

    // Auto-accept Cloudflare Tunnel domains
    if (origin && origin.includes('.trycloudflare.com')) {
      console.log(`[CORS] ✅ Auto-accepting Cloudflare Tunnel: ${origin}`);
      return callback(null, true);
    }

    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      // TEMPORARY: Let the public live-stats endpoint handle its own CORS via response headers
      // Other public routes may also need open CORS in the future
      callback(null, true);  // ⚠️ accepte TOUTES les origins inconnues
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'Accept', 'X-API-Key', 'sentry-trace', 'baggage'],
});
```

**Conséquence directe** : le `else` retourne `callback(null, true)`. **N'importe quelle origin** (y compris attaquante) est autorisée, avec `credentials: true`. La whitelist `API_CORS_ORIGIN` n'a en pratique aucun effet bloquant.

### 1.2 Variables d'environnement

| Variable | Défaut | Fichier | Note |
|---|---|---|---|
| `API_CORS_ORIGIN` | `http://localhost:3001` | [src/config/validation.ts](../../../attendee-ems-back/src/config/validation.ts#L23) | Liste séparée par virgules |
| Dev (`.env`) | `http://localhost:5173,http://localhost:3000` | [.env](../../../attendee-ems-back/.env#L22) | Front Vite + autre |
| Docker (`.env.docker`) | `http://localhost:5173,http://localhost:3000` | [.env.docker](../../../Public-Form-Logger/.env.docker#L22) | |
| Prod | `https://yourdomain.com,https://www.yourdomain.com` | [README.md](../../README.md#L989) | Exemple |

Une seule variable existe. Pas de distinction `dev/staging/prod` au niveau du nom.

### 1.3 WebSocket (Socket.io)

[src/websocket/websocket.gateway.ts](../../../attendee-ems-back/src/websocket/websocket.gateway.ts#L14-L19) :

```ts
@WebSocketGateway({ cors: { origin: '*', credentials: true } })
```

**Wildcard `*` avec `credentials: true`** : configuration incohérente (les navigateurs rejettent cette combinaison) — fonctionne probablement car Socket.io fait son propre handshake, mais c'est **hors-norme**.

### 1.4 Routes publiques actuelles (sans JWT)

| Route | Fichier | Besoin CORS externe ? |
|---|---|---|
| `GET /api/public/events/:token` | [public.controller.ts](../../../attendee-ems-back/src/modules/public/public.controller.ts) | Oui (page d'inscription externe) |
| `GET /api/public/events/:token/attendee-types` | idem | Oui |
| `GET /api/public/events/:token/tables` | idem | Oui |
| `GET /api/public/events/:token/live-stats` | idem (L44-L48) | **Oui** — `Access-Control-Allow-Origin: *` posé manuellement |
| `POST /api/public/events/:token/register` | idem | Oui (formulaire public) |
| `POST /api/public/events/:token/login` | [attendee-access.controller.ts](../../../attendee-ems-back/src/modules/public/attendee-access/attendee-access.controller.ts) | Oui (webinaire) |
| `GET/POST /api/public/events/:token/me\|logout\|checkin\|online-stats` | idem | Oui (webinaire intégré) |
| `GET /api/badges/:id/pdf` (`@Public()`) | [badges.controller.ts](../../../attendee-ems-back/src/modules/badges/badges.controller.ts#L115) | Plutôt non (téléchargement direct, pas XHR cross-origin) |
| `GET /api/badges/:id/image` (`@Public()`) | idem L155 | Idem |
| `GET /api/qrcode/:registrationId` | [qrcode.controller.ts](../../../attendee-ems-back/src/modules/email/qrcode.controller.ts) | Non (image dans email — pas de preflight) |
| `POST /api/auth/login`, `POST /api/auth/password-reset` | [auth.controller.ts](../../../attendee-ems-back/src/auth/auth.controller.ts) | Uniquement depuis front EMS |
| `GET /api/health` | [health.controller.ts](../../../attendee-ems-back/src/health/health.controller.ts) | Non |

### 1.5 Consommateurs de l'API

| Client | Comment il appelle | Origin envoyé | Credentials |
|---|---|---|---|
| `attendee-ems-front` (Vite SPA) | `fetch` avec `credentials: 'include'` ([rootApi.ts](../../../attendee-ems-front/src/services/rootApi.ts#L60-L61)) | Oui (domaine du SPA) | **Oui** |
| `attendee-ems-mobile` (Expo / RN) | Axios, `Authorization: Bearer …` ([apiUrl.ts](../../../attendee-ems-mobile/src/config/apiUrl.ts#L3-L21)) | **Non** (pas de navigateur) | Non |
| `attendee-ems-print-client` (Electron) | `fetch` Node — pas d'`Origin` envoyé ([main.js](../../../attendee-ems-print-client/main.js#L6)) | **Non** | Non |
| `Public-Form-Logger` | Backend Node | **Non** | Non |
| Swagger UI `/api/docs` | Servi par le back lui-même → **same-origin** | Same | – |
| Postman / curl | – | **Non** | – |

**Conclusion clé** : seuls les **navigateurs** envoient un `Origin`. Mobile / Electron / Postman / Swagger ne sont **pas concernés** par CORS. Restreindre CORS ne casse **ni mobile**, **ni print client**, **ni Swagger/Postman**.

---

## 2. Pourquoi la config actuelle ?

Hypothèses documentées dans les commentaires de `main.ts` :

1. **Débloquer live-stats embarqué** dans un dashboard externe.
2. **Permettre l'inscription publique** depuis n'importe quel site partenaire (formulaire iframe).
3. **Tunnels Cloudflare** en dev/demo (`*.trycloudflare.com` change à chaque session).
4. **Éviter de bloquer les apps mobiles** — confusion : mobile n'envoie pas d'Origin.

Le `callback(null, true)` fallback est un **contournement** des points 1+2 sans avoir mis en place un mécanisme distinct pour les routes publiques.

---

## 3. Risques

| Risque | Niveau | Détail |
|---|---|---|
| **CSRF amplifié** | 🔴 Haut | `credentials: true` + origin `*` (de facto) → tout site peut faire `fetch('https://api…', { credentials: 'include' })` et lire la réponse pour un utilisateur connecté. |
| **Vol de session JWT cookie** | 🔴 Haut | Si refresh token en cookie HttpOnly + CORS ouvert, un site malveillant peut déclencher des actions authentifiées (mutations PUT/DELETE). |
| **Exfiltration de données privées** | 🔴 Haut | Toute route protégée (registrations, attendees, exports) est lisible depuis n'importe quel domaine si la session est active. |
| **WebSocket `origin: '*'`** | 🟠 Moyen | Permet à n'importe quel site de monter un Socket.io et recevoir événements temps réel. |
| **Reverse tabnabbing / phishing** | 🟠 Moyen | Sous-domaines clients ou pages partenaires malveillantes peuvent invoquer l'API. |
| **Audit / conformité** | 🟠 Moyen | OWASP A05:2021 (Security Misconfiguration), bonne pratique RGPD pour données nominatives. |

---

## 4. Points à vérifier avant migration

- [ ] Confirmer la liste des domaines **prod** réellement utilisés par le front EMS.
- [ ] Confirmer le domaine **staging**.
- [ ] Identifier toutes les pages externes/iframes qui appellent réellement `live-stats` et `register` (analytics, dashboard client, etc.).
- [ ] Vérifier que `Public-Form-Logger` n'appelle **pas** d'endpoint navigateur (sinon ajouter son domaine).
- [ ] Confirmer que le print-client n'a pas été migré vers une version qui envoie `Origin` (build Electron récent peut le faire selon la config `webPreferences`).
- [ ] Vérifier que les **webhooks n8n** entrent par `X-API-Key`, pas par navigateur (CORS sans impact).
- [ ] Lister les sous-domaines `*.trycloudflare.com` réellement actifs et leur usage (dev only ? demo client ?).

---

## 5. Architecture cible proposée

### 5.1 Niveau 1 — Whitelist statique par environnement (à implémenter)

#### 5.1.1 Variables d'environnement

| Variable | Type | Exemple dev | Exemple staging | Exemple prod |
|---|---|---|---|---|
| `NODE_ENV` | enum | `development` | `staging` | `production` |
| `CORS_ALLOWED_ORIGINS` | csv | `http://localhost:5173,http://localhost:3000` | `https://staging.ems.example.com` | `https://app.ems.example.com,https://admin.ems.example.com` |
| `CORS_ALLOWED_ORIGIN_PATTERNS` | csv regex | `^https?://(localhost\|127\.0\.0\.1)(:\d+)?$,^https://[a-z0-9-]+\.trycloudflare\.com$` | (vide) | (vide) |
| `CORS_PUBLIC_ROUTES_OPEN` | bool | `true` | `true` | `true` |
| `CORS_ALLOW_NO_ORIGIN` | bool | `true` | `true` | `true` (mobile/Electron/Postman) |

> Renommer `API_CORS_ORIGIN` → `CORS_ALLOWED_ORIGINS` (avec **compat ascendante** : si `CORS_ALLOWED_ORIGINS` absent, fallback sur `API_CORS_ORIGIN`).

#### 5.1.2 Logique cible (pseudocode, **non implémentée ici**)

```ts
function buildCorsOriginFn(env, cfg) {
  const exact = new Set(cfg.allowedOrigins);
  const patterns = cfg.allowedOriginPatterns.map(p => new RegExp(p));
  const isProd = env === 'production';

  return (origin, cb) => {
    // 1. No origin → mobile / Electron / Postman / curl
    if (!origin || origin === 'null') {
      return cb(null, cfg.allowNoOrigin);
    }
    // 2. Exact match
    if (exact.has(origin)) return cb(null, true);
    // 3. Regex match (utilisé surtout en dev pour *.trycloudflare.com)
    if (patterns.some(re => re.test(origin))) return cb(null, true);
    // 4. Refus explicite — pas de fallback `true`
    if (isProd) {
      // log structuré (Sentry warn) avec origin pour audit
      return cb(new Error(`CORS: origin not allowed: ${origin}`), false);
    }
    // En dev, log et refuser (mais sans throw bruyant)
    return cb(null, false);
  };
}
```

**Différence clé vs actuel** :
- Plus de `callback(null, true)` en fallback.
- Wildcard `.trycloudflare.com` **uniquement** via `CORS_ALLOWED_ORIGIN_PATTERNS` → contrôlé par env (dev/staging only).
- En prod : refus explicite + log.

#### 5.1.3 Routes publiques (live-stats, register, badge PDF)

Garder une politique **séparée** pour les endpoints réellement embarqués :

- Soit un **décorateur `@PublicCors()`** qui ajoute manuellement `Access-Control-Allow-Origin: *` **sans `credentials`** (comme déjà fait pour live-stats).
- Soit un **interceptor** appliqué aux contrôleurs `public/*` qui pose les headers CORS publics. La CORS globale ne s'applique alors qu'aux routes navigateur authentifiées.

> Important : `Access-Control-Allow-Origin: *` **n'est compatible qu'avec `credentials: omit`** côté client. C'est ok pour live-stats / badge PDF qui n'utilisent pas de cookie.

#### 5.1.4 WebSocket

Remplacer `origin: '*'` par la **même fonction** que la CORS HTTP (réutilisable), via `WebSocketGateway({ cors: { origin: buildCorsOriginFn(...) } })`.

### 5.2 Niveau 2 — Whitelist dynamique en base (**à NE PAS implémenter maintenant**)

Pour le futur, si des **clients tiers** veulent embarquer le formulaire/live-stats sur leur propre domaine :

#### 5.2.1 Modèle Prisma proposé

```prisma
model AllowedOrigin {
  id          String   @id @default(cuid())
  origin      String   @unique           // "https://client-x.com"
  orgId       String?                    // null = global
  scope       OriginScope @default(PUBLIC) // PUBLIC | FULL
  enabled     Boolean  @default(true)
  note        String?
  createdAt   DateTime @default(now())
  createdBy   String?
  expiresAt   DateTime?

  @@index([enabled, scope])
}

enum OriginScope {
  PUBLIC   // ne donne accès qu'aux routes /api/public/*
  FULL     // routes authentifiées (rare, valider au cas par cas)
}
```

#### 5.2.2 Service avec cache

```ts
@Injectable()
export class AllowedOriginCache {
  private cache: Map<string, AllowedOrigin> = new Map();
  private lastRefresh = 0;
  private ttlMs = 60_000; // 1 min

  async isAllowed(origin: string, scope: OriginScope): Promise<boolean> {
    await this.refreshIfStale();
    const entry = this.cache.get(origin);
    return !!entry && entry.enabled && (entry.scope === 'FULL' || scope === 'PUBLIC');
  }

  invalidate() { this.lastRefresh = 0; } // appelé après écriture admin
}
```

- TTL 60 s → délai max ~1 min avant qu'un nouveau domaine soit actif **sans redémarrage**.
- Invalidation immédiate via `invalidate()` au moment de l'`INSERT`/`UPDATE`.
- Lecture **uniquement mémoire** au preflight → pas de DB hit par requête.

#### 5.2.3 Pourquoi pas maintenant

- Pas de besoin client confirmé.
- Ajoute une surface d'attaque admin (qui peut ajouter un domaine ?).
- Le `.env` couvre 100 % des cas actuels une fois la whitelist correcte.

---

## 6. Limites du `.env`

| Limite | Impact | Atténuation |
|---|---|---|
| Modifier la liste nécessite **redémarrage** | Coupure brève, OK en prod si rare | Process scellé à chaque deploy → cohérent |
| Pas de scoping par org | Tout le monde voit les mêmes origins | Acceptable tant que clients = même tenant front |
| Pas de UI admin | Manip par DevOps | OK pour MVP, motive le passage au niveau 2 plus tard |
| Pas d'expiration | Origins anciens restent | Audit manuel périodique |

---

## 7. Recommandation finale

**Implémenter le Niveau 1 immédiatement**, garder le Niveau 2 comme évolution future.

### Cibles concrètes :

1. **Production** : liste **stricte** dans `CORS_ALLOWED_ORIGINS` (front prod + admin éventuel). Pas de patterns. Refus explicite des origins inconnues.
2. **Staging** : liste stricte du domaine staging.
3. **Dev** : liste localhost + pattern `^https://[a-z0-9-]+\.trycloudflare\.com$` autorisé **par pattern env**, pas en dur dans le code.
4. **Routes publiques** : décorateur `@PublicCors()` posant `ACAO: *` **sans credentials** — sépare la politique "API auth" (stricte) de "API publique embarquable" (ouverte mais sans cookie).
5. **WebSocket** : même fonction que HTTP, pas de `*` en prod.
6. **Pas de breaking change pour mobile / Electron / Postman / Swagger** : `CORS_ALLOW_NO_ORIGIN=true`.

---

## 8. Plan de migration progressif

### Phase 0 — Préparation (analyse seule, ce document)
- [x] Audit code + clients
- [ ] Validation business : qui doit pouvoir embarquer quoi ?
- [ ] Récup liste domaines prod/staging confirmée

### Phase 1 — Ajout config (non bloquant)
- [ ] Introduire `CORS_ALLOWED_ORIGINS` (alias de `API_CORS_ORIGIN`)
- [ ] Introduire `CORS_ALLOWED_ORIGIN_PATTERNS`
- [ ] Mettre à jour `.env.example`, `.env.docker`, `.env`, README
- [ ] **Aucun changement de comportement** : le fallback `callback(null, true)` reste.

### Phase 2 — Mode "shadow" (observabilité)
- [ ] Logger toutes les origins **refusées par la whitelist** (mais encore acceptées) avec compteur Sentry / structured log.
- [ ] Laisser tourner 1–2 semaines en prod.
- [ ] Analyser : aucune origin légitime non listée ne doit apparaître.

### Phase 3 — Bascule `@PublicCors()`
- [ ] Créer le décorateur + interceptor pour les routes publiques (`public/*`, `badges/:id/pdf`, `badges/:id/image`, `qrcode/*`).
- [ ] Vérifier que `live-stats` continue de fonctionner sans le fallback global.

### Phase 4 — Activation strict mode
- [ ] Remplacer `callback(null, true)` (else branch) par `callback(new Error(...), false)`.
- [ ] Surveiller Sentry 48 h.

### Phase 5 — WebSocket
- [ ] Appliquer la même fonction à `WebSocketGateway`.
- [ ] Tester reconnexion clients front en prod.

### Phase 6 — Nettoyage
- [ ] Retirer le pattern `trycloudflare` du code → uniquement via env dev/staging.
- [ ] Documenter la politique dans `attendee-context-hub`.

### Phase 7 (futur) — Niveau 2 dynamique
- [ ] Si besoin confirmé : table `AllowedOrigin` + cache.

---

## 9. Checklist de tests

### 9.1 Tests fonctionnels manuels (par phase)

| # | Cas | Attendu dev | Attendu prod |
|---|---|---|---|
| 1 | Front EMS sur domaine whitelisté → `GET /api/registrations` avec cookie | 200 + `ACAO` | 200 + `ACAO` |
| 2 | Front sur domaine **non** whitelisté → même requête | (dev) 200 + warn | **CORS refusé** |
| 3 | Mobile (RN) → `POST /api/auth/login` (pas d'Origin) | 200 | 200 |
| 4 | Print client Electron → `GET /api/print-queue/pending` | 200 | 200 |
| 5 | Postman → n'importe quelle route protégée avec Bearer | 200 | 200 |
| 6 | Swagger `/api/docs` → "Try it out" | 200 | 200 |
| 7 | Site tiers `https://evil.com` → `fetch('/api/registrations', { credentials: 'include' })` | (dev) 200 + warn | **CORS refusé** |
| 8 | Site tiers → `GET /api/public/events/:token/live-stats` | 200 + `ACAO: *` | 200 + `ACAO: *` |
| 9 | Site tiers → `POST /api/public/events/:token/register` | 200 | 200 (avec rate-limit) |
| 10 | Email client → `<img src="…/api/qrcode/:id">` | image OK | image OK |
| 11 | Tunnel `xxx.trycloudflare.com` → front dev | 200 | refusé |
| 12 | WebSocket depuis front whitelisté | connect OK | connect OK |
| 13 | WebSocket depuis site tiers | (dev) OK | **refusé** |
| 14 | Preflight OPTIONS sur `/api/registrations` depuis domaine non listé | (dev) 204 | **403/CORS error** |

### 9.2 Tests automatisés à ajouter
- [ ] e2e CORS : un fichier `test/cors.e2e-spec.ts` envoyant Origin headers variés et vérifiant `Access-Control-Allow-Origin`.
- [ ] Unit test sur la fonction `buildCorsOriginFn`.

### 9.3 Régression à surveiller
- [ ] Aucune erreur CORS dans la console front prod après bascule.
- [ ] Pas de chute du nombre de connexions WebSocket.
- [ ] Pas d'augmentation des 401/403 (signe d'un blocage CORS mal interprété).

---

## 10. Contraintes respectées

- ✅ Mobile : pas d'Origin → toujours accepté via `allowNoOrigin`.
- ✅ Print client Electron : pas d'Origin → idem.
- ✅ Swagger / Postman : same-origin / pas d'Origin → idem.
- ✅ Routes publiques (`/api/public/*`, `badges/*/pdf|image`, `qrcode/*`) : politique CORS ouverte **dédiée** via décorateur, sans `credentials`.
- ✅ Production : refus explicite des origins inconnues.
- ✅ Aucun refactor hors scope CORS.

---

## 11. Références

- OWASP A05:2021 — Security Misconfiguration
- MDN — [HTTP access control (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- NestJS — [Enabling CORS](https://docs.nestjs.com/security/cors)
- Code source actuel : [src/main.ts](../../../attendee-ems-back/src/main.ts#L45-L77), [src/websocket/websocket.gateway.ts](../../../attendee-ems-back/src/websocket/websocket.gateway.ts#L14-L19), [src/config/validation.ts](../../../attendee-ems-back/src/config/validation.ts#L23)
