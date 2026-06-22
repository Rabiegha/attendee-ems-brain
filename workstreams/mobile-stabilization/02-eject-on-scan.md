# 02 — Éjection à l'ouverture du scan

## Symptôme

Quand l'utilisateur **ouvre l'écran de scan**, l'app **revient à l'écran Events list** (le premier écran après le login), sans crash, sans retour au login.

> ⚠️ "Éjection" dans ce workstream = **retour forcé à l'Events list**. Ce n'est pas un crash ni un logout.

## Statut

🔍 À reproduire et instrumenter. **Ne pas corriger avant d'avoir isolé la cause.**

---

## Hypothèse principale — identifiée dans le code

> 🎯 Cause probable confirmée par lecture du code. À valider en reproduction.

Le mécanisme d'éjection est le **même que dans 01** (NavigationContainer conditionnel + AppState listener). La différence : c'est la **dialog de permission caméra** qui déclenche l'AppState transition.

### Cas 1 — Permission caméra (première ouverture)

`ScanScreen.tsx` demande la permission au montage si `permission === null` :
```javascript
useEffect(() => {
  if (!permission) {
    await requestPermission(); // → ouvre la dialog système
  }
}, [permission, requestPermission]);
```
- La dialog système fait passer l'app en état `inactive`.
- Quand l'utilisateur tape "Autoriser", l'app repasse `active`.
- AppState listener : `inactive → active` → `restoreSessionThunk` → `bootstrapOrgContextThunk` → si switch déclenché → éjection.
- Après la première autorisation : `permission.granted = true` → plus de dialog → plus de transition `inactive` → **plus d'éjection**. C'est pour ça que ça ne se produit qu'une fois.

### Cas 2 — Session scan après hard restart (aléatoire)

Après un hard restart :
1. `restoreSessionThunk` (non-silent) s'exécute → peut déclencher un switch si `lastOrgId` diffère.
2. La navigation démarre sur Events list (initial route).
3. L'utilisateur navigue vers EventInner → Session Scan.
4. Si pendant cette navigation l'app passe brièvement en `inactive` (notification, veille rapide) → nouveau cycle AppState → possible switch → éjection.

Pour les scans de **sessions spécifiques** (Google Cloud Summit) : probable que certaines sessions déclenchaient un 403 (pas de permission sur cette session précise). Le handler `handleTenantDriftThunk` se déclenche sur 403 org-scoped, refetche `/my-orgs`, et si l'utilisateur n'est plus membre → `clearOrgContext()` → `currentOrgId = null` → prochain bootstrap voit mono-org avec `currentOrgId === null` → `switchOrgThunk` → éjection.

### À confirmer en logs

```
[ScanScreen] Requesting camera permission...
[AuthSlice] bootstrapOrgContext - Auto-switching to last org: ...
[AppContent] App returned to foreground, verifying session...
[AppContent] Tenant drift candidate (403 on org-scoped route): ...
```

---

## Plan de repro

1. Se connecter, naviguer vers l'écran de scan.
2. Tester avec permission caméra : accordée / refusée / jamais demandée.
3. Tester sur dev client ET sur build EAS (le natif peut différer).
4. Capturer la stack trace (logs / Sentry si présent).

## Observations

**Repro live 2026-06-10 (Pixel 10a, dev client sur back prod) :**
- À chaque foreground, les logs montrent en boucle :
  ```
  [AppContent] App returned to foreground, verifying session + refreshing orgs...
  bootstrapOrgContext - state: { lastOrgId: null, isCurrentOrgStillValid: true }
  ```
  `isCurrentOrgStillValid: true` → jamais de switch → **aucune éjection** sur background/scan/veille.
- Un **403 tenant-drift** a firé en live à 14:55 sur `/events/ab4b5fcf…/stats` → **toujours pas d'éjection** : `handleTenantDrift` a vu l'org encore membre → no-op. C'est donc un 403 **permission-scoped**, pas un drift d'org.

## Cause racine

Le `lastOrgId: null` est un **faux indice** : c'est l'état normal (écrit uniquement par `switchOrgThunk`). Le vrai coupable historique = **`currentOrgId = null` au cold-start** dans les anciens builds (non persisté/dérivé) → `isCurrentOrgStillValid = false` → branche auto-switch → reset nav → éjection.

Corrigé par : dérivation de `currentOrgId` depuis `organization.id` au `persist/REHYDRATE`, synchro post-restauration (`0cb2ba8`), déblocage splash (`abdd52d`). Sur HEAD, plus reproductible.

Le 403 `/events/:id/stats` est **distinct** : scope `events.read:assigned` (VIEWER/PARTNER/HOSTESS) + user non assigné à l'event via `EventAccess` → `ForbiddenException` dans le service. L'app affichait les stats depuis le **cache offline** malgré le 403.

## Décision de correction

- Code : déjà corrigé sur HEAD. **À livrer par build store** (la prod ne reçoit pas l'OTA, cf. README).
- Métier (403 stats) : assigner l'user à l'event **ou** rôle en `events.read:org`. Pas de permission à ajouter.
- À valider : repro ciblée du **vrai déclencheur** (multi-org switch / drift où l'org est réellement retirée).
