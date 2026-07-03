# Refactor Mobile — Expo / React Native (`attendee-ems-mobile`)

> **Branche à créer :** `Refactore-authorization-v1`
> **Particularité :** déploiement OTA via EAS — penser au versioning de la build native si breaking change.

## Audit préalable

```bash
cd attendee-ems-mobile
grep -rn "isPlatform\|\.mode\b\|'platform'" src/ --include="*.ts" --include="*.tsx"
grep -rn "auth/login\|auth/switch-org\|auth/me/orgs\|auth/me/ability\|auth/policy\|auth/available-orgs" src/
grep -rn "tenantRoles\|TenantUserRole\|TenantRole" src/ types/
```

Lister les occurrences AVANT de coder.

## Spécificités mobile vs web

| Sujet | Différence |
|---|---|
| Stockage token | `expo-secure-store` au lieu de `localStorage` |
| Switch org | Souvent moins fréquent → vérifier qu'il marche quand même |
| Bypass root | Probablement pas utilisé sur mobile (root = web admin), mais à vérifier |
| Cache permissions | Si offline-first, vérifier que les permissions cache sont invalidées au switch |

## Paliers Mobile

### Palier M1 — Adaptation types et clients API

- [ ] Mettre à jour `types/auth.ts` (ou équivalent) pour ne plus dépendre de `mode`/`isPlatform`
- [ ] Adapter `src/services/api/auth.ts` (ou équivalent)
- [ ] Si `auth/available-orgs` est appelé → migrer vers `auth/me/orgs`
- [ ] Vérifier que le refresh token survit au switch d'organisation

### Palier M2 — Logique métier

Probables endroits à auditer :
- `App.tsx` / navigation racine
- Hook d'auth (`useAuth`, `useUser`)
- Écran de sélection d'organisation (s'il existe)
- Écrans protégés par permissions

Stratégie identique au front web : tout `mode === 'platform'` ou `isPlatform` → `isRoot` ou check de permission précis.

### Palier M3 — State global et persistence

- [ ] Store mobile (Zustand/Redux/Context) : retirer `mode` / `isPlatform`
- [ ] Vérifier `secureStore.getItem('auth')` — invalider la cache si format change
- [ ] Migration douce : si un user a un objet auth en `secureStore` avec l'ancien format, le tolérer (ne pas crasher au démarrage)

### Palier M4 — Tests sur device

- [ ] Test sur **iOS** (simulateur + un device réel) — login + navigation + switch org
- [ ] Test sur **Android** (émulateur + un device réel)
- [ ] Test offline → online (cache permissions toujours valide)
- [ ] Test "logout puis re-login avec un autre user" → pas de fuite d'état

---

## 🎨 UX multi-organisation (transverse aux paliers M1-M4)

Sur mobile, la place est limitée → les patterns diffèrent un peu du web mais le besoin est le même : un user multi-org doit pouvoir choisir son org au login, et changer d'org sans relogger.

### M-UX.1 — Comportement post-login

À la réception du `LoginResponse`, idem web :

| Cas | Comportement |
|---|---|
| **0 org** | Écran d'erreur "Aucune organisation associée. Contactez un administrateur." + bouton logout |
| **1 org** | Auto-sélection silencieuse → switch → home tab |
| **2+ orgs** | **Écran de sélection** (liste scrollable) → tap → switch → home tab |

### M-UX.2 — Écran de sélection d'organisation (post-login)

**Composant à créer/adapter :** `src/screens/SelectOrganizationScreen.tsx`

```
┌────────────────────────┐
│ Bonjour Rachid 👋      │
│                        │
│ Choisissez une org :   │
│                        │
│ ┌────────────────────┐ │
│ │ 🏢 Choyou          │ │
│ │ Administrator    > │ │
│ └────────────────────┘ │
│ ┌────────────────────┐ │
│ │ 🏢 Acme Corp       │ │
│ │ ROOT             > │ │
│ └────────────────────┘ │
│ ┌────────────────────┐ │
│ │ 🏢 System          │ │
│ │ ROOT             > │ │
│ └────────────────────┘ │
└────────────────────────┘
```

**Interactions :**

- Liste verticale `FlatList` (scrollable si beaucoup d'orgs)
- Tap sur un item → loader → switch → home tab
- Pull-to-refresh pour recharger `me/orgs` (utile si invitation acceptée entre temps)

### M-UX.3 — Switcher d'org dans l'app (décision retenue)

**Contrainte :** le composant `Header` existant (cf [Header.tsx](../../../../../../../attendee-ems-mobile/src/components/ui/Header.tsx)) a une structure figée — zones fixes 48px à gauche/droite, titre centré, hauteur 68px. **Pas question d'y injecter un sélecteur global** (ça casserait tous les screens).

**Décision :**

1. **Switcher principal = écran `Settings → Mes organisations`** — pour les users multi-org, entrée dédiée dans les paramètres :

   ```
   Settings
   ┌─────────────────────────────┐
   │ ‹     Paramètres            │
   ├─────────────────────────────┤
   │  👤  Mon profil          ›  │
   │  🏢  Mes organisations   ›  │  ← visible UNIQUEMENT si multi-org
   │       Choyou (active)       │
   │  🔔  Notifications        › │
   │  ──────────────────────     │
   │  🚪  Déconnexion            │
   └─────────────────────────────┘
   ```

   L'écran `MyOrganizationsScreen` reprend la même UI que l'écran de sélection post-login (M-UX.2), avec un radio sur l'org active.

2. **Raccourci unique = `EventsListScreen`** (cf [src/screens/Events/EventsListScreen.tsx](../../../../../../../attendee-ems-mobile/src/screens/Events/EventsListScreen.tsx)) — pour les users multi-org, le **titre du header** est remplacé par le **nom de l'org**, tappable pour ouvrir une bottom sheet de switch :

   ```
   Mono-org (aucun changement)        Multi-org (titre = nom d'org)
   ┌─────────────────────────┐        ┌─────────────────────────┐
   │ ‹   Mes événements   ⋮  │        │ ‹      Choyou ▾      ⋮  │
   ├─────────────────────────┤        ├─────────────────────────┤
   │ [À venir][En cours][...] │        │ [À venir][En cours][...] │
   └─────────────────────────┘        └─────────────────────────┘
                                              ↓ tap
                                      ┌─────────────────────────┐
                                      │      ━━━━━━              │
                                      │  Changer d'organisation │
                                      │  ●  Choyou              │
                                      │  ○  Acme Corp   [ROOT]  │
                                      │  ○  Festival 2026       │
                                      └─────────────────────────┘
   ```

   C'est **le SEUL screen** où l'org est dans le header (page d'accueil après login = celle où on revient le plus souvent). Sur tous les autres screens, le header reste tel quel (titre du screen).

**Composants à créer :**

- `src/screens/Settings/MyOrganizationsScreen.tsx` (nouvel écran)
- `src/components/OrganizationSwitcherSheet.tsx` (bottom sheet réutilisé par EventsListScreen + MyOrganizationsScreen)
- Modification de `EventsListScreen.tsx` ligne ~48 : remplacer `title={t('events.title')}` par un titre conditionnel multi-org → nom d'org + chevron tappable

**Règle absolue :**

> ⚠️ **Aucun changement visible pour un user mono-org.** Pas d'entrée Settings → Mes organisations, pas de chevron sur le titre EventsListScreen, pas de modal modifié. L'app doit être **strictement identique** à aujourd'hui pour eux.

Implémentation : un hook `useIsMultiOrg()` qui retourne `true` si `me/orgs.length > 1`, et chaque comportement multi-org est gardé derrière ce hook.

### M-UX.4 — Action au switch (pareil que web)

1. `POST /auth/switch-org { orgId }`
2. Stocker le nouveau token dans `secureStore` (remplace l'ancien)
3. **Invalider toutes les caches** React Query / Zustand / Redux
4. **Recharger les permissions** via `GET /me/ability`
5. Reset du stack de navigation (`navigation.reset({ routes: [{ name: 'Home' }] })`) — important pour ne pas garder une page de l'ancienne org
6. Toast / haptic feedback : "Vous êtes sur **Acme Corp**"

⚠️ **Reset de navigation obligatoire** sinon l'utilisateur pourrait revenir en arrière sur une page de l'ancienne org via le bouton retour (Android surtout).

### M-UX.5 — Mémorisation de la dernière org

Idem web mais en `secureStore` :

```ts
await SecureStore.setItemAsync('lastOrgId', orgId);
```

Au prochain login :
- Si `lastOrgId` ∈ `me/orgs` → auto-sélection (skip écran M-UX.2)
- Sinon → écran M-UX.2 normal

### M-UX.6 — Indicateur visuel d'org active

**Récap de la décision (cf M-UX.3) :**

| Élément | Mono-org | Multi-org |
|---|---|---|
| Header de `EventsListScreen` | `Mes événements` (titre normal) | **`Choyou ▾`** (tappable → bottom sheet switch) |
| Header des autres screens | inchangé | inchangé |
| Settings → entrée "Mes organisations" | absente | présente |
| Modals de confirmation (suppression, envoi, etc.) | **inchangés** | **inchangés** |
| Page d'ajout / création | **inchangée** | **inchangée** |
| Couleur de thème par org | ❌ écartée | ❌ écartée |

> ⚠️ **Aucun rappel "Action sur Choyou" dans les modals**, même pour les multi-org. Le nom d'org dans le header de `EventsListScreen` + l'écran Settings → Mes organisations suffisent. On ne pollue pas chaque modal d'action.
>
> Raison : le user multi-org sait sur quelle org il est (visible dans le header de la page d'accueil), et on évite de différencier l'UI des modals entre mono et multi-org → plus simple à maintenir, moins de QA, code modal identique pour tout le monde.

**Conclusion :** le seul indicateur visuel permanent est le **titre du header sur `EventsListScreen`** (en multi-org uniquement).

### M-UX.7 — Cas spécial root sans rôle

Idem web :

- Badge "ROOT" coloré à côté du nom d'org
- Pas d'erreur permissions vides
- **Bandeau d'alerte** en haut "Mode super-administrateur" (couleur d'avertissement, dismissible par session)

### M-UX.8 — Spécificités mobile à ne pas oublier

- [ ] **Background → foreground** : si l'app a été en arrière-plan plus de X minutes, recharger `me/orgs` au retour
- [ ] **Token expiré** pendant un switch → catch 401, déclencher le refresh, retry une fois
- [ ] **Notifications push** : si une notif vient d'une autre org, proposer le switch automatique au tap
- [ ] **Deep linking** : si un lien profond pointe vers une ressource d'une autre org, switch automatique avant d'ouvrir
- [ ] **Performance** : la liste d'orgs est petite (1-10 généralement) → pas besoin de pagination

### M-UX.9 — V2 : push temps réel via WebSocket

> 📌 **À implémenter en V2, pas dans le scope V1.**

Idem web (cf F-UX.7-bis) : abonnement aux events `org-membership-added` / `org-membership-removed` / `org-role-changed` côté back, avec invalidation immédiate de la liste d'orgs et des permissions sans avoir à attendre un retour foreground ou un refresh manuel.

Spécificité mobile : la connexion WebSocket doit être pausée en background et reprise au foreground (économie batterie + données mobiles).

### Checklist UX multi-org mobile

- [ ] 0 org : message d'erreur + logout
- [ ] 1 org : auto-sélection
- [ ] 2+ orgs : écran de sélection au login
- [ ] Pull-to-refresh sur l'écran de sélection
- [ ] **Hook `useIsMultiOrg()` créé et utilisé partout où l'UI diffère**
- [ ] **`EventsListScreen` : titre = nom d'org tappable UNIQUEMENT si multi-org**
- [ ] **`Settings` : entrée "Mes organisations" UNIQUEMENT si multi-org**
- [ ] **Mono-org : zéro changement visible (header, modals, pages d'ajout)**
- [ ] Aucun modal de confirmation modifié pour ajouter "Action sur X" (décision M-UX.6)
- [ ] Mémorisation `lastOrgId` en secureStore
- [ ] Cache invalidée au switch
- [ ] Stack de navigation reset au switch
- [ ] Haptic feedback / toast au switch
- [ ] Badge spécifique pour rôle ROOT
- [ ] Recharge `me/orgs` au foreground après X min en background
- [ ] Deep links / notifs push gèrent le multi-org

---

## Critères d'acceptation

- [ ] `tsc --noEmit` vert
- [ ] `npm run lint` vert
- [ ] Tests Detox (si configurés) verts (voir `06-TESTS-UNITAIRES-FONCTIONNELS-E2E.md`)
- [ ] Build EAS preview OK pour iOS et Android
- [ ] Smoke test manuel sur les 3 rôles principaux

## Stratégie de déploiement mobile

⚠️ Le mobile **ne se déploie pas comme le web**. Trois cas :

1. **Pure JS change (90% du refactor)** → OTA update via EAS Update, instantané
2. **Lib native ajoutée/upgradée** → nouveau build EAS, soumission stores (App Store + Play Store), délai 1 à 7 jours
3. **Breaking change format token / API** → version mineure, gérer le mapping de l'ancien format pendant N jours

Pour ce refactor → **cas 1 ou 3** selon si on tolère l'ancien format auth en cache. Recommandé : **cas 3 avec tolérance**, pour ne forcer personne à se reconnecter.

## Ne PAS faire dans ce palier

- ❌ Bumper la version major sans raison (les stores sont lents)
- ❌ Casser la rétrocompat du `secureStore` sans migration
- ❌ Pousser une OTA update avant que le back de prod ait les endpoints attendus
