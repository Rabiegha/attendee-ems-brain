# Architecture

Décisions et schémas d'**archi stables** qui concernent plusieurs apps. Vient de l'inbox `[archi]`.

> **Pas de découpe fait / en cours / à faire ici** : ce dossier garde des **décisions stables**, pas des tâches.

> Quand une décision devient une **règle définitive d'un repo**, la déplacer dans le `context/` du repo concerné.

| Sujet | Apps | Note |
|---|---|---|
| [Organization Plan & Subscription Model](organization-plan-subscription-model.md) | back + produit | Le plan/abonnement appartient à l'**organisation**, jamais au user (billing, quotas par tenant). Cible conceptuelle, aucun code modifié. |
| [Platform Admin Access Model](platform-admin-access-model.md) | back + front | Séparation stricte **tenant access** vs **platform access** : rôles, guards et routes distincts. Cible conceptuelle, aucun code modifié. |
