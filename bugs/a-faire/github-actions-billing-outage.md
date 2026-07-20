# Incident — GitHub Actions bloqué (facturation compte)

> **Statut** : Confirmé, pas encore traité par l'utilisateur.
> **Découvert** : 2026-07-20, ~16h00, en pleine session d'activation Gotenberg (démo LFD2026).
> **App** : `attendee-ems-back` (mais concerne le compte/organisation GitHub entier, pas un repo précis).
> **Repo de référence** : [attendee-ems-back](../../../attendee-ems-back/)

## Symptôme

Tous les workflows GitHub Actions échouent en 2-4 secondes (jamais réellement exécutés), sur
CI comme sur CD :

```
The job was not started because recent account payments have failed or your spending
limit needs to be increased. Please check the 'Billing & plans' section in your settings
```

Confirmé sur `gh run view <id>` (annotations des jobs `build-image`, `e2e-tests`,
`lint-typecheck`, `unit-tests`).

## Étendue constatée (2026-07-20)

Historique `gh run list --limit 10` sur `attendee-ems-back` :

| Run | Workflow | Event | Heure | Résultat |
|---|---|---|---|---|
| 29754322802 | CD staging | workflow_run | 15:15 | ✅ success (dernier succès) |
| 29755115841 | CI | pull_request | 15:25 | ✅ success |
| 29755447004 | CI | push | 15:30 | ✅ success (dernier succès) |
| 29755851294 | CD staging | workflow_run | 15:35 | ❌ failure |
| 29756155453 | CI | pull_request | 15:39 | ❌ failure |
| 29756754246 | CI | pull_request | 15:47 | ❌ failure |
| 29757377801 | CI | push | 15:55 | ❌ failure |
| 29757387042 | CD staging | workflow_run | 15:56 | ⏭️ skipped |
| 29758018710 | CI | pull_request | 16:04 | ❌ failure |
| 29758255528 | CI | pull_request | 16:07 | ❌ failure |

→ Bascule nette entre 15:30 (dernier succès) et 15:35 (premier échec). Tout ce qui a suivi
est cassé, y compris le **CD staging** (donc plus aucun déploiement auto possible tant que
ce n'est pas réglé).

## Impact concret

- PR [#56](https://github.com/Rabiegha/attendee-ems-back/pull/56) (volume secrets Gotenberg)
  et [#57](https://github.com/Rabiegha/attendee-ems-back/pull/57) (scripts diagnostic
  Gotenberg) : CI rouge, mais échec de facturation, pas de code — safe à merger sans CI
  une fois confirmé, ou à re-run une fois la facturation réglée.
- Aucune CI/CD ne peut tourner sur AUCUN repo de l'organisation tant que ce n'est pas résolu.

## Action requise (hors scope agent — accès facturation nécessaire)

Aller dans **GitHub → Settings → Billing & plans** du compte/organisation `Rabiegha`,
régler le moyen de paiement en échec ou augmenter la limite de dépense Actions.

## Prochaine étape

Une fois réglé par l'utilisateur : re-run les checks sur PR #56 et #57
(`gh pr checks 56 --watch`, idem 57), et vérifier que le CD staging repart normalement
sur le prochain push.
