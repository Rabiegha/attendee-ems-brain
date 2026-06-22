# 01 — Diagnostic éjection hard restart

## Objectif

Reproduire de façon fiable l'éjection de l'application après redémarrage complet du process Android.

## Définition

Un **hard restart** signifie que le process natif de l'application est tué, puis l'application est relancée depuis zéro.

Sur Android, le test fiable est :

```bash
adb shell am force-stop com.choyou.attendeemobile
adb shell monkey -p com.choyou.attendeemobile 1
```

> Raccourci npm : `npm run android:hard-restart`

---

## Symptôme observé

Au hard restart, l'utilisateur se retrouve sur l'**écran Events list** (premier écran post-login) au lieu de retrouver l'écran où il était.

> "Éjection" dans ce workstream = retour forcé à l'Events list. Ce n'est pas un crash ni un retour au login.

---

## Statut

🔍 À reproduire et instrumenter. **Ne pas corriger avant d'avoir isolé la cause.**

---

## Hypothèse principale — identifiée dans le code

> 🎯 Cause probable confirmée par lecture du code. À valider en reproduction.

### Le mécanisme : NavigationContainer conditionnel

Dans `AppNavigator.tsx`, le `NavigationContainer` n'est **pas toujours monté**. Quand la condition suivante est vraie, il est **démonté** et remplacé par un spinner :
```javascript
if (isRestoringAuth || showBootstrapSplash || isWaitingForOrgContext) {
  return <ActivityIndicator />; // NavigationContainer est démonté !
}
```
Quand il se **remonte**, la navigation repart de l'écran initial = **Events list**. C'est ça, "l'éjection".

### Le déclencheur : AppState listener

Dans `AppContent.tsx`, à **chaque** transition `inactive/background → active` :
```javascript
AppState.addEventListener('change', (nextAppState) => {
  if (appState.current.match(/inactive|background/) && nextAppState === 'active') {
    dispatch(restoreSessionThunk({ silent: true })); // ← ici
  }
});
```
Cela inclut : retour depuis une autre app, réveil de l'écran, **fermeture d'une dialog système** (ex : permission caméra).

### La chaîne qui démonte le NavigationContainer

`restoreSessionThunk` → `bootstrapOrgContextThunk` → dans certains cas → `switchOrgThunk`
- `switchOrgThunk.pending` → `isSwitchingOrg = true`
- `isWaitingForOrgContext = isAuthenticated && !needsOrgSelection && (isSwitchingOrg || !hasOrgContext)` → **true**
- NavigationContainer **démonté** → remonte sur Events list

### Quand `switchOrgThunk` est-il déclenché par bootstrap ?

Dans `bootstrapOrgContextThunk` :
- **Mono-org** : si `currentOrgId !== onlyOrg.orgId` (peut arriver si `currentOrgId` est null/périmé).
- **Multi-org** : si `lastOrgId` (lu en SecureStore) ≠ `currentOrgId` actuel → auto-switch.

### Pourquoi c'est aléatoire ?

- Dépend de l'état de `currentOrgId` et du `lastOrgId` en SecureStore au moment du foreground.
- Si `currentOrgId` correspond déjà → pas de switch → pas d'éjection.
- Si réseau lent → la fenêtre où `isSwitchingOrg = true` dure plus longtemps → éjection visible.
- Debounce de 10s : si l'utilisateur revient au foreground moins de 10s après, pas de restore → pas d'éjection.

### À confirmer en logs

```
[AuthSlice] bootstrapOrgContext - Auto-switching to last org: ...
[AuthSlice] switchOrgThunk - Switching to org: ...
```
Si ces lignes apparaissent dans `adb logcat` au moment de l'éjection → cause confirmée.

---

## Prérequis

- Avoir une **development build** ou **preview build** installée sur le téléphone (pas Expo Go).
- ADB configuré et fonctionnel. Vérifier avec :
  ```bash
  adb devices
  ```
- Le téléphone doit apparaître dans la liste (pas `unauthorized`).
- L'application doit être installée sur le téléphone.
- Metro peut être lancé avec :
  ```bash
  npx expo start --dev-client
  ```

---

## Procédure de test

1. Lancer l'app et se connecter.
2. Naviguer jusqu'à l'état à tester (ex : rester sur le dashboard d'un event, ou aller sur le scan).
3. Vider les logs :
   ```bash
   npm run android:clear-logs
   ```
4. Déclencher le hard restart :
   ```bash
   npm run android:hard-restart
   ```
5. Observer si l'app :
   - s'ouvre normalement sur le bon écran ✅
   - revient à l'Events list (l'éjection qu'on cherche à reproduire)
   - revient au login
   - reste bloquée sur le splash screen
   - se ferme brutalement
   - affiche une erreur JS
   - crash à l'ouverture du scan
6. En cas d'éjection ou comportement anormal, capturer les logs :
   ```bash
   npm run android:logs:errors
   ```

---

## Journal de reproduction

| Date | Build | Device | État avant restart | Résultat après restart | Dernier écran visible | Logs utiles | Hypothèse |
|---|---|---|---|---|---|---|---|
| | | | | | | | |

---

## Checklist d'analyse

- [ ] L'app crash avant l'écran login.
- [ ] L'app crash après réhydratation auth.
- [ ] L'app revient au login alors qu'un token devrait exister.
- [ ] L'app reste bloquée sur splash.
- [ ] L'app ouvre un écran sans données nécessaires.
- [ ] L'app crash seulement à l'ouverture du scan.
- [ ] Les logs montrent une erreur `ReactNativeJS`.
- [ ] Les logs montrent une erreur `AndroidRuntime / FATAL EXCEPTION`.

---

## Cause racine

<!-- à remplir une fois confirmée -->

## Décision de correction

<!-- à remplir -->
