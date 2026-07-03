# Palier 4 — Retrait définitif des champs HTTP dépréciés

> **Statut :** ⏳ À faire **après** que front + mobile soient déployés sans dépendre de ces champs.
> **Branche :** `Refactore-authorization-v1` (suite) ou nouvelle branche `palier-4-cleanup-http`.
> **Risque :** breaking change sur les contrats HTTP — ne pas faire seul, deploy synchronisé.

## Pourquoi un palier dédié ?

Pendant Paliers 3a/3b/3c, on a gelé les champs `mode` et `isPlatform` dans les réponses HTTP avec valeurs sentinelle (`'tenant'` et `false`). Ce gel sert de pont :
1. Le back est déjà débarrassé de la logique platform en interne
2. Le front et le mobile peuvent encore consommer l'API legacy sans crasher
3. Une fois front + mobile mergés et déployés sans dépendre de ces champs, on peut les retirer définitivement

Le faire trop tôt = casser le front en prod.
Le faire trop tard = jamais.

## Pré-requis avant d'attaquer ce palier

- [x] Palier 3c back mergé et déployé (commit `c6d8e7a`)
- [ ] Front refactor v1 mergé et **déployé en production**
- [ ] Mobile refactor v1 mergé et **diffusé via OTA / store**
- [ ] Au moins **7 jours** d'observation prod sans erreur côté front/mobile (laisser le temps aux apps mobiles de se mettre à jour)
- [ ] Confirmation grep côté front et mobile :

```bash
cd attendee-ems-front && grep -rn "isPlatform\|response\.mode" src/  # → vide
cd attendee-ems-mobile && grep -rn "isPlatform\|response\.mode" src/ # → vide
```

## Items à supprimer

### 1. Réponses HTTP

| Endpoint | Champ à retirer |
|---|---|
| `POST /auth/login` | `mode: 'tenant'` dans la réponse top-level |
| `POST /auth/switch-org` | `mode: 'tenant'` dans la réponse |
| `GET /auth/me/orgs` | `isPlatform: false` sur chaque item |
| `GET /me/ability` | `mode: 'tenant'` top-level + `role.isPlatform: false` |

### 2. Types DTO/interfaces

- `src/auth/interfaces/jwt-payload.interface.ts::AvailableOrg` : retirer `isPlatform`
- `src/auth/interfaces/jwt-payload.interface.ts::UserAbility` : retirer `mode` si encore présent
- `src/platform/authz/core/types.ts::UserAbility` : retirer `mode` et `role.isPlatform`

### 3. Code de construction des réponses

- `src/auth/auth.service.ts::login` : ne plus injecter `mode: 'tenant'` dans le retour
- `src/auth/auth.service.ts::switchOrg` : pareil
- `src/auth/auth.service.ts::getAvailableOrgs` : ne plus mettre `isPlatform: false` sur les items
- `src/platform/authz/adapters/http/controllers/me-ability.controller.ts` : retirer `mode` et `role.isPlatform` de la réponse

### 4. Documentation

- Mettre à jour les docs API (Swagger / OpenAPI s'il y en a)
- Mettre à jour `docs/Refactore-autorization/` pour acter le retrait
- Notifier les équipes qui consomment l'API (si applicable)

## Validation

- [ ] `npx tsc --noEmit` vert
- [ ] `bash scripts/test-palier2-curl.sh` adapté (retirer assertions sur `mode`/`isPlatform`)
- [ ] Smoke test web sur staging (si dispo) ou local front+back
- [ ] Smoke test mobile sur staging
- [ ] Aucune erreur Sentry post-deploy

## Plan de déploiement spécifique Palier 4

⚠️ **Synchronisation impérative** :

1. Préparer le PR back Palier 4
2. Vérifier que le front en prod ne lit plus les champs (Sentry, logs nginx, surveillance)
3. Vérifier que < X% des sessions mobiles utilisent une version trop ancienne (analytics)
4. Merger + deploy back **en heures creuses**
5. Surveiller les logs 30 minutes
6. Si erreur → rollback immédiat (image Docker précédente, pas de migration DB à reverter)

## Rollback

Le Palier 4 ne touche **pas** à la DB. Le rollback est un simple redéploiement de l'image Docker précédente. Aucun backup spécifique nécessaire.

## Ne PAS faire dans ce palier

- ❌ Combiner ce palier avec d'autres changements (DB, RBAC, features)
- ❌ Le déployer en même temps que le front si le front n'est pas déjà en prod depuis quelques jours
- ❌ Sauter l'étape de vérification grep côté front/mobile
