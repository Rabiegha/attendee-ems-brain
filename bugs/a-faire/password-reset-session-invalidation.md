# Reset mot de passe — faut-il invalider les sessions actives ?

> **App :** back (+ front / mobile impactés) · **Type :** revue sécurité / question ouverte
> **Status :** À investiguer — comportement à décider, **pas encore tranché**
> **Date :** 2026-07-02

## Question

Quand un utilisateur **réinitialise son mot de passe** alors qu'il est **déjà connecté**
(navigateur front, ou app mobile), que doit-il se passer pour les sessions existantes ?

- Faut-il le **déconnecter** partout (invalider les tokens en cours) ?
- Faut-il garder la session courante active et ne couper que **les autres** appareils ?
- Ou ne rien faire (comportement actuel à vérifier) ?

## Pourquoi c'est un sujet

Un reset de mot de passe est souvent déclenché **parce que** l'utilisateur soupçonne un
accès non autorisé. Si le reset ne coupe pas les sessions existantes (refresh tokens toujours
valides), un attaquant déjà connecté **garde l'accès** malgré le changement de mot de passe.
C'est le comportement attendu de la plupart des apps : *changer le mot de passe = fermer les
autres sessions*.

## À vérifier (comportement actuel)

- Que fait aujourd'hui le flow reset côté back : `attendee-ems-back/src/auth/` (service reset
  password) — révoque-t-il les `refresh_tokens` de l'utilisateur ? (a priori **non**).
- La table `refresh_tokens` : y a-t-il un moyen de révoquer en masse par `userId` ?
- Côté front/mobile : la session courante survit-elle (token d'accès encore valide jusqu'à
  expiration) même après changement du mot de passe ?

## Pistes (à décider, ne pas implémenter sans validation)

1. **Révoquer tous les refresh tokens** de l'utilisateur au reset → déconnexion de tous les
   appareils, y compris celui qui a fait le reset (re-login demandé). Le plus sûr.
2. **Révoquer tout sauf la session courante** → l'utilisateur qui reset reste connecté, les
   autres appareils sont éjectés. Meilleur compromis UX/sécurité, mais plus complexe (il faut
   identifier « la session courante »).
3. Ajouter un champ `passwordChangedAt` sur le user et invalider les tokens émis avant cette
   date (vérif dans la stratégie JWT / au refresh).

## Lien

- Rotation refresh token déjà sensible ici : [2026-07-01-eject-mobile-refresh-rotation-499.md](./2026-07-01-eject-mobile-refresh-rotation-499.md).
- Revue CORS / sécurité endpoints : [cors-origin-security-review.md](./cors-origin-security-review.md).

## Règle

Analyse / décision d'abord. **Ne pas corriger** sans avoir tranché le comportement voulu
(déconnexion totale vs partielle) avec un accord explicite.
