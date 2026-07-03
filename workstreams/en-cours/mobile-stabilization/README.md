# Workstream — Diagnostic & stabilisation mobile

## Pilote

```
App:      attendee-ems-mobile
Status:   Active
Priority: High
```

**Objectif :** Rendre l'app mobile stable en supprimant les **éjections** (retour forcé vers le login / crash) dans deux scénarios :
1. **Hard restart** de l'app.
2. Ouverture de l'**écran de scan**.

**Approche :** d'abord **diagnostiquer** (reproduire + isoler la cause racine), seulement ensuite corriger. Pas de correction à l'aveugle.

---

## État actuel

| Partie | Sujet | Status |
|---|---|---|
| [01](01-eject-hard-restart.md) | Éjection foreground/wake/hard restart | ✅ Non reproductible sur HEAD — cause environnement (OTA) |
| [02](02-eject-on-scan.md) | Éjection à l'ouverture du scan / session scan | ✅ Non reproductible sur HEAD — cause environnement (OTA) |

**Prochaine action :** publier un **nouveau build store** (pas un OTA) pour livrer les fixes en prod. Voir conclusion ci-dessous.

---

## 🔑 Conclusion du diagnostic (2026-06-10)

Repro tentée en live sur device physique (Pixel 10a, dev client connecté au **back de prod** `api.attendee.fr`). **Impossible de reproduire l'éjection** sur le code actuel (`HEAD = abdd52d`).

**Faits durs établis :**

1. **Backend identique dev/prod.** `.env` + tous les profils `eas.json` → `https://api.attendee.fr/api`. Ce n'est pas le serveur. Logs confirment : mêmes données (Google Cloud Summit 2026, 5512 inscriptions).

2. **Le chemin sain tourne en boucle sans éjecter.** À chaque foreground :
   ```
   App returned to foreground...
   bootstrapOrgContext - state: { lastOrgId: null, isCurrentOrgStillValid: true }
   ```
   `isCurrentOrgStillValid: true` → jamais de `switchOrgThunk` → jamais d'éjection. Le déclencheur réel est `isCurrentOrgStillValid = false` (drift 403 / multi-org), **pas** un simple background/scan/veille.

3. **La prod est figée sur un bundle incompatible OTA — mécanisme PRÉCIS (corrigé le 2026-06-10).**

   La preuve n'est PAS un simple écart de fingerprint, c'est un **mélange de deux policies `runtimeVersion`** :

   | | `runtimeVersion` | Type | Source |
   |---|---|---|---|
   | Binaire **store prod** (1.1.7, 3 juin) | `"1.1.7"` | chaîne de version | `eas build:list` |
   | **Tous les OTA** (12 derniers) | `0faf20a2…`, `05f87e2a…`, `b44dd915…` | hash fingerprint | `eas update:list --branch production` |

   Règle expo-updates : un OTA n'est livré qu'à un binaire de `runtimeVersion` **strictement égal**. `"1.1.7"` (chaîne) ne peut **jamais** égaler un hash de 40 caractères. Le build store a été fait avec une policy **version-string**, puis la policy a été **basculée vers `fingerprint`** dans `app.json`. → **aucun OTA ne peut atteindre la prod 1.1.7.** Elle tourne en boucle sur son bundle embarqué du 3 juin. **Certitude 100% (preuve par les runtimeVersions, pas par déduction).**

**Conséquence opérationnelle :** les correctifs anti-éjection (`abdd52d`, `fd9a6c9`, `81e5283`, `0cb2ba8`) existent en local mais **ne peuvent PAS arriver en prod par OTA**. → **Nouveau build store (EAS production) requis.**

---

## 🧠 Cause racine de l'éjection (élucidée 2026-06-10)

Le bug n'est **pas** `lastOrgId`. `lastOrgId: null` est **normal** (écrit uniquement par `switchOrgThunk`, donc absent tant qu'on n'a pas switché manuellement). Le vrai coupable est **`currentOrgId = null` dans le state Redux** lors de la **réhydratation redux-persist après un kill** (cold-start), dans les anciens builds qui ne persistaient/dérivaient pas ce champ.

> ⚠️ **Deux `currentOrgId` distincts — ne pas confondre :**
> - **`currentOrgId` du JWT** (dans le token) → lu par le **backend** (`jwt.strategy.ts`). **Toujours présent** tant que le token est valide. Le mobile n'envoie que `Authorization: Bearer <token>`, aucun header d'org.
> - **`currentOrgId` du state Redux** (`auth.slice.ts`) → utilisé **uniquement** par la navigation/bootstrap front.
>
> C'est pour ça que l'app « marchait quand même » : le `null` Redux **n'impacte jamais les appels API** (le back a l'org via le token). Il ne casse **que** la logique de navigation → éjection. Symptôme typique : « il y avait l'erreur mais je voyais quand même les stats ».

> 🎲 **Bug intermittent**, pas systématique : après un login frais `currentOrgId` est set. Le `null` n'apparaît qu'au combo cold-start + réhydratation sans `currentOrgId` (+ parfois multi-org/lastOrgId) → d'où le « aléatoire ».


```
currentOrgId = null  (non persisté/dérivé au cold-start)
  → isCurrentOrgStillValid = false
  → branche auto-switch (switchOrgThunk)
  → reset navigation
  → ÉJECTION vers Events list
```

Les fixes locaux dérivent désormais `currentOrgId` depuis `organization.id` persisté (`persist/REHYDRATE`), le synchronisent après restauration (`0cb2ba8`) et débloquent le splash (`abdd52d`). → `currentOrgId` n'est plus jamais null → `isCurrentOrgStillValid: true` → plus de switch → **plus d'éjection**. Confirmé par les logs live (boucle `isCurrentOrgStillValid: true`).

## 🔒 Le 403 `/events/:id/stats` (élucidé 2026-06-10) — N'EST PAS une éjection + BUG ROOT

Le 403 a firé en live (14:55) **sans éjecter** (`handleTenantDrift` a vu l'org encore valide → no-op). Ce n'est **pas une permission manquante**.

**Cause racine confirmée : le compte était `root`, et les utils de scope ignorent `isRoot`.**

Chaîne pour un compte root :
1. JWT minimal → `req.user.role` est **`undefined`** (`jwt.strategy.ts` ne porte pas le rôle).
2. Guard RBAC : `if (isRoot) allow()` → **bypass OK**, la requête atteint le controller.
3. **MAIS** `resolveEventReadScope(req.user)` (resolve-event-scope.util.ts) **ne regarde jamais `isRoot`** : role undefined + permissions sans `events.read:any` → fallback final → scope **`assigned`**.
4. `findOne` scope `assigned` + event non assigné → `ForbiddenException` (403).

→ **Un compte root bypasse le GUARD mais PAS la couche service.** `resolveEventReadScope` / `resolveRegistrationScope` / `resolveEventWriteScope` ignorent `isRoot` → root traité comme un user sans rôle → scope `assigned` → 403. L'app affichait les stats depuis le **cache offline**.

**Fix backend (à faire) :** ajouter `if (user.isRoot === true) return 'any';` en tête des 3 utils `resolve*Scope`. Le guard pose déjà `req.user.isRoot`.
Fix métier alternatif (user non-root) : assigner l'user à l'event (`EventAccess`) **ou** rôle en `events.read:org`.

## 🧾 Dettes ouvertes

- 🔴 Stratégie `runtimeVersion` à standardiser (version-string vs fingerprint).
- 🔴 Prod non patchable par OTA → rebuild store obligatoire.
- � **`resolve*Scope` ignorent `isRoot`** → un compte root reçoit 403 sur events non assignés (incohérence guard/service). Ajouter `if (user.isRoot) return 'any'`.
- �🟠 Fail-open dans `resolveEventReadScope` (« no permissions → any ») → passer fail-closed.
- 🟠 `EXPO_PUBLIC_PRINTNODE_API_KEY` en clair dans `eas.json`.
- 🟠 socket.io abandonne après 5 tentatives, pas de reconnexion au retour réseau.
- 🟡 Cache affiche stats périmées malgré 403 (pas d'indicateur UX).
- 🟡 `console.log` debug laissés en prod backend (`resolveEventReadScope`, `findOne`).

**Reste à faire pour clore :**
- [ ] Publier un build store `production` 1.1.8 (livre le nouveau fingerprint + les fixes), versionCode 6.
- [ ] Repro ciblée du vrai déclencheur : compte **multi-org** + changement d'org, ou **403 tenant drift** forcé (org réellement retirée), pour valider que les fixes couvrent bien ce cas.

---

## Découpage

Le chantier est volontairement découpé en parties indépendantes. On peut traiter 01 et 02 séparément. On ajoute `03-…` etc. si d'autres instabilités apparaissent (à capturer via [INBOX.md](../../../workspace-rabie/INBOX.md)).

---

## Contexte technique repéré

**Le mécanisme d'éjection est identifié dans le code.**
Voir section "Hypothèse principale" dans [01](01-eject-hard-restart.md) et [02](02-eject-on-scan.md) pour le détail.

Fichiers clés :
- `src/AppContent.tsx` — AppState listener + `restoreSessionThunk`
- `src/navigation/AppNavigator.tsx` — rendu conditionnel du `NavigationContainer`
- `src/store/auth.slice.ts` — `bootstrapOrgContextThunk`, `switchOrgThunk`, `isWaitingForOrgContext`
- `src/screens/Scan/ScanScreen.tsx` — `useCameraPermissions`, demande de permission au montage

---

## Définition de "terminé"

- [ ] Cause racine identifiée pour chaque éjection.
- [ ] Correctif appliqué et validé sur build EAS.
- [ ] Aucune éjection sur 5 hard restarts consécutifs.
- [ ] Aucune éjection sur 5 ouvertures de scan consécutives.
- [ ] Apprentissages consignés dans [learnings/](../../../learnings/README.md).

---

## Liens

- [Cerveau / méthode](../../../README.md)
- [NOW](../../../workspace-rabie/NOW.md)
- Mémoire repo mobile (EAS, .env, dette) — voir notes de l'agent.
