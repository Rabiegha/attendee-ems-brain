# Refactor Front — Web app (`attendee-ems-front`)

> **Branche :** `Refactore-authorization-v1`
> **Objectif :** aligner le front sur la nouvelle API authz (un seul mode, plus de notion `platform`), préparer le retrait des champs HTTP dépréciés (Palier 4 back).

## Audit préalable (à exécuter en premier)

Avant de toucher au code, il faut **savoir ce que le front consomme aujourd'hui** :

```bash
cd attendee-ems-front
# Champs API dépréciés
grep -rn "isPlatform\|\.mode\b\|'platform'\|'tenant'" src/ --include="*.ts" --include="*.tsx"
# Endpoints concernés
grep -rn "auth/login\|auth/switch-org\|auth/me/orgs\|auth/me/ability\|auth/policy\|auth/available-orgs" src/
# Anciens noms de relations Prisma s'il y en a en types partagés
grep -rn "tenantRoles\|TenantUserRole\|TenantRole" src/
```

Lister chaque occurrence dans une checklist avant de coder.

## Paliers Front

### Palier F1 — Adaptation des types et clients API

**Objectif :** stopper la dépendance aux champs dépréciés sans casser la prod actuelle.

- [ ] Créer/mettre à jour les types DTO dans `src/types/auth.ts` (ou équivalent) :
  - `LoginResponse` : ne plus exposer `mode` côté composants (le garder en réception, ne pas l'utiliser)
  - `AvailableOrg` : retirer `isPlatform` du type interne ; si nécessaire le garder optionnel le temps de la migration
  - `UserAbility` : pareil pour `mode` et `role.isPlatform`
- [ ] Adapter les clients HTTP (probablement dans `src/services/` ou `src/api/`) pour ne plus lire `mode`/`isPlatform`
- [ ] Si l'endpoint `auth/available-orgs` était utilisé : remplacer par `auth/me/orgs`
- [ ] Vérifier que `auth/switch-org` est bien appelé en `POST` avec `{ orgId }` dans le body

**Tests :**
- Build TypeScript clean (`tsc --noEmit`)
- Login → token reçu, navigation OK
- Switch d'organisation → token rafraîchi, redirection sur la bonne org

### Palier F2 — Suppression usage `mode` / `isPlatform` dans la logique métier

**Objectif :** purger toute condition qui s'appuyait sur `mode === 'platform'` ou `isPlatform === true`.

Probables endroits :
- Composant racine / `App.tsx` — initialisation post-login
- Garde de routes (`PrivateRoute`, `RequirePermission`)
- Sélecteur d'organisation (header / drawer)
- Pages admin platform (à supprimer si plus de notion)

Stratégie :
- Tout ce qui était `if (user.isPlatform)` ou `if (mode === 'platform')` → conditionner sur `isRoot` ou sur la présence d'une permission précise (`organizations.read:any`)
- Si une page entière était réservée au mode platform et n'a plus de sens → la supprimer + retirer ses routes

**Tests :**
- Login en tant que root → toujours bypass des restrictions (visible UI)
- Login en tant qu'admin org → restrictions normales
- Login en tant qu'agent → restrictions par scope OK

### Palier F3 — État global (Redux/Zustand/Context)

**Objectif :** retirer `mode` et `isPlatform` du state global front.

- [ ] Identifier le store auth (`authSlice`, `useAuthStore`, `AuthContext`…)
- [ ] Retirer les champs `mode` et `isPlatform` du state shape
- [ ] Adapter les sélecteurs (`selectIsPlatform` → soit suppression, soit `selectIsRoot`)
- [ ] Vérifier qu'aucune persistence (localStorage, sessionStorage) ne stocke ces valeurs

### Palier F4 — Polish UI

- [ ] Retirer mentions textuelles "platform" si visibles à l'utilisateur (badges, libellés)
- [ ] Vérifier que le sélecteur d'org affiche correctement les orgs où l'utilisateur n'a pas de rôle (cas root)
- [ ] Si page profil affichait "mode: platform" → retirer

---

## 🎨 UX multi-organisation (transverse aux paliers F1-F4)

C'est le **vrai changement utilisateur** du refactor. Avant, beaucoup d'users étaient mono-org ou platform. Maintenant, n'importe qui peut être membre de plusieurs orgs (multi-tenant), et un root voit toutes les orgs. L'UI doit le rendre fluide.

### F-UX.1 — Comportement post-login

À la réception du `LoginResponse`, le front doit décider de la suite selon le nombre d'orgs renvoyées par `GET /auth/me/orgs` :

| Cas | Comportement attendu |
|---|---|
| **0 org** | Page d'erreur "Aucune organisation associée à votre compte. Contactez un administrateur." |
| **1 org** | Auto-sélection silencieuse → `POST /auth/switch-org` → redirection sur le dashboard |
| **2+ orgs** | **Écran de sélection d'organisation** (liste cliquable) → choix → switch → dashboard |

⚠️ Pour root : il verra toujours **3+ orgs** (Choyou, Acme, System), donc passera systématiquement par l'écran de sélection. Penser à pouvoir mémoriser le dernier choix (cf F-UX.4).

### F-UX.2 — Écran de sélection d'organisation (post-login)

**Composant à créer/adapter :** `src/pages/SelectOrganization.tsx` (ou équivalent)

Maquette minimale :

```
┌─────────────────────────────────────────────┐
│  Bonjour Rachid 👋                          │
│  Sélectionnez l'organisation à utiliser :   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 🏢 Choyou                            │   │
│  │    Administrator                     │   │
│  └─────────────────────────────────────┘   │
│  ┌─────────────────────────────────────┐   │
│  │ 🏢 Acme Corp                         │   │
│  │    ROOT (accès super-admin)          │   │
│  └─────────────────────────────────────┘   │
│  ┌─────────────────────────────────────┐   │
│  │ 🏢 System                            │   │
│  │    ROOT (accès super-admin)          │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Champs affichés par item :** `orgName`, `role`, indicateur visuel pour root.
**Action au clic :** `POST /auth/switch-org { orgId }` → stocker le nouveau token → `navigate('/dashboard')`.

### F-UX.3 — Sélecteur d'org dans le header (toujours accessible)

Une fois sur l'app, l'utilisateur multi-org doit pouvoir changer d'organisation **sans se relogger**. Le sélecteur vit en permanence dans le header / topbar.

**Composant à créer/adapter :** `src/components/layout/OrganizationSwitcher.tsx`

Comportement :

- Affiche en permanence l'org active (nom + petit badge rôle)
- Clic → dropdown avec la liste de toutes les orgs (issue de `GET /auth/me/orgs`)
- Item actuellement sélectionné : checkmark / surbrillance
- Items autres : cliquables → switch
- Si user mono-org : afficher le nom mais **désactiver** le dropdown (pas de switch possible)

Maquette dropdown :

```
┌──────────────────────────────┐
│ 🏢 Choyou (Administrator) ▼  │  ← clic ici
├──────────────────────────────┤
│ ✓ Choyou                     │
│   Administrator              │
│                              │
│   Acme Corp                  │
│   ROOT                       │
│                              │
│   System                     │
│   ROOT                       │
├──────────────────────────────┤
│   Gérer mes organisations →  │  (optionnel)
└──────────────────────────────┘
```

**Action au clic sur un autre item :**

1. `POST /auth/switch-org { orgId }`
2. Stocker le nouveau token (remplace l'ancien)
3. **Invalider toutes les caches** (React Query / SWR / Zustand) — les données de l'ancienne org ne sont plus pertinentes
4. **Recharger les permissions** via `GET /me/ability`
5. Rediriger vers le dashboard de la nouvelle org (ou rester sur la page courante si elle existe pour la nouvelle org)
6. Toast / feedback : "Vous êtes maintenant sur **Acme Corp**"

⚠️ **Piège** : si l'utilisateur était sur `/events/123` (un event de Choyou) et switche sur Acme, l'event 123 n'existe pas pour Acme → 404. Solution : toujours rediriger sur `/dashboard` au switch (plus simple et plus sûr).

### F-UX.4 — Mémorisation de la dernière org

Pour éviter à l'utilisateur de re-sélectionner son org à chaque login :

- Au switch réussi, persister `lastOrgId` en `localStorage`
- Au login suivant, si `lastOrgId` est dans la liste de `me/orgs` → auto-sélection (skip écran F-UX.2)
- Si `lastOrgId` n'est plus dans la liste (révocation d'accès) → afficher l'écran F-UX.2 normalement
- Bouton "Changer d'organisation" toujours dispo dans le header (cf F-UX.3)

### F-UX.5 — Indicateur visuel d'org active

**Décision (alignée avec la décision mobile M-UX.6) :**

| Élément | Mono-org | Multi-org |
|---|---|---|
| Nom d'org dans le header | absent ou statique | **toujours visible + tappable** (cf F-UX.3) |
| Switcher dans le menu user | absent | présent |
| Modals de confirmation (suppression, envoi, etc.) | **inchangés** | **inchangés** |
| Pages de création / édition | **inchangées** | **inchangées** |
| Couleur de thème par org | ❌ écartée | ❌ écartée |

> ⚠️ **Aucun rappel "Action sur Choyou" injecté dans les modals**, même en multi-org.
>
> Raison : le nom d'org est toujours visible dans le header (F-UX.3), c'est suffisant. On évite de différencier le code des modals entre mono et multi-org → moins de complexité, code et QA identiques pour tout le monde.

**Règle absolue :**

> 🔒 **Aucun changement visible pour un user mono-org.** Pas de switcher, pas de header modifié, pas de modal différent. L'app doit être strictement identique à aujourd'hui pour eux.
>
> Implémentation : un hook `useIsMultiOrg()` qui retourne `true` si `me/orgs.length > 1`, et chaque comportement multi-org est gardé derrière ce hook.

### F-UX.6 — Cas spécial root sans rôle dans une org

Quand un root switche sur une org où il n'a aucun `UserRole` :

- ✅ `me/orgs` la liste avec `role: 'ROOT'`
- ✅ `me/ability` répond 200 (grants vides) — fix Palier 3c
- ✅ Tous les endpoints répondent 200 grâce au bypass

L'UI doit :

- Afficher un **badge spécifique** "ROOT" (couleur distincte) à côté du nom d'org pour rappeler que c'est un mode super-admin
- Pas afficher d'erreur permissions vides
- **Bandeau d'alerte** en haut de page : "Vous accédez à cette organisation en mode super-administrateur" (couleur d'avertissement, dismissible par session)

### F-UX.7 — Rafraîchissement de la liste d'orgs

Si une nouvelle invitation est acceptée par l'utilisateur (ou un admin l'ajoute à une org), le `GET /auth/me/orgs` doit être rappelé pour mettre à jour le sélecteur :

- Au login systématiquement
- Après acceptation d'invitation
- **Refetch automatique** : la query `useGetMyOrgsQuery()` est utilisée dans le `Header` (composant monté en permanence) → bénéficie du `refetchOnFocus: true` déjà configuré dans [rootApi.ts](../../../../../../../attendee-ems-front/src/services/rootApi.ts) → mise à jour automatique quand l'utilisateur revient sur l'onglet
- ❌ Pas de bouton "Recharger" manuel (le refetch automatique au focus suffit)
- ❌ Pas de refresh périodique (inutile, ajoute du bruit réseau)

**Drift detection (à gérer dans le composant Header) :**

- Si `availableOrgs.length` augmente → toast discret : *"Vous avez maintenant accès à **\<nouvelle org\>**"*
- Si l'org active disparaît de `availableOrgs` (révocation d'accès) → forcer un re-switch sur la première org dispo, ou rediriger vers `/select-org`
- Si la liste devient vide → forcer logout

### F-UX.7-bis — V2 : push temps réel via WebSocket / SSE

> 📌 **À implémenter en V2, pas dans le scope V1.**

Le mécanisme `refetchOnFocus` (F-UX.7) couvre 95 % des cas mais a un défaut : tant que l'utilisateur reste actif sur la même page sans changer d'onglet, il ne voit pas les nouvelles orgs.

**Solution V2 :**

- Le back émet un event WebSocket / SSE sur changement de membership :
  - `org-membership-added` : `{ userId, orgId, orgName, role }` → push aux sockets de ce userId
  - `org-membership-removed` : `{ userId, orgId }` → idem
  - `org-role-changed` : `{ userId, orgId, newRole }` → idem
- Le front (probablement dans le `AbilityProvider` ou un nouveau `RealtimeProvider`) écoute ces events et :
  - Invalide le tag RTK Query `MyOrgs` → refetch instantané
  - Si org active concernée : invalide aussi `Policy` → recharge les permissions sans refresh page
  - Affiche un toast immédiat (pas besoin de drift detection)
- Bénéfice secondaire : utilisable aussi pour les changements de permissions (modification de rôle, ajout de permission) sans reload manuel

**Pré-requis back V2 :** un service de pub/sub (déjà présent côté `websocket/` mais à étendre pour publier ces events authz spécifiques).

### Checklist UX multi-org

- [ ] 0 org : message d'erreur clair
- [ ] 1 org : auto-sélection (UX fluide)
- [ ] 2+ orgs : écran de sélection au login
- [ ] Switcher header toujours dispo si multi-org
- [ ] Switcher header désactivé si mono-org (mais nom visible)
- [ ] Mémorisation `lastOrgId` en `localStorage`
- [ ] Cache invalidée au switch
- [ ] Permissions rechargées au switch
- [ ] Redirection sur dashboard au switch (pas garder URL courante)
- [ ] Toast de confirmation au switch
- [ ] Badge spécifique pour rôle ROOT
- [ ] Nom de l'org active visible en permanence
- [ ] Refresh de la liste après acceptation invitation

---

## Critères d'acceptation

- [ ] `tsc --noEmit` vert
- [ ] `npm run lint` vert
- [ ] Tests unitaires verts (`npm test`)
- [ ] Tests Playwright e2e verts (voir `04-TESTS-E2E-INTEGRATION.md`)
- [ ] Smoke test manuel sur les 3 rôles principaux (root, admin, agent)
- [ ] Aucune référence à `isPlatform` ni `\.mode\b` dans `src/` (sauf typage de réception API si encore nécessaire)

## Compatibilité descendante

Pendant le développement front, le back garde les champs dépréciés en réponse HTTP (Palier 3b). Une version intermédiaire du front qui ignore ces champs continue de fonctionner avec la prod actuelle du back. Une fois le front mergé et déployé, on peut lancer le **Palier 4 back** (retrait définitif des champs HTTP).

## Ne PAS faire dans ce palier

- ❌ Modifier le back en parallèle
- ❌ Changer la logique RBAC côté front (les permissions viennent toujours du JWT)
- ❌ Supprimer les composants admin sans valider qu'ils ne sont plus utilisés
