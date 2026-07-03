# Organization Plan & Subscription Model

> **Statut** : Documentation architecture / Cible conceptuelle. **Aucun code modifié.**
> **Public** : Architecture, Backend, Produit

---

## 1. Objectif

Documenter **pourquoi** et **comment** le plan/subscription doit appartenir à l'**organisation**, jamais au user.

Raisons :

- Les **données métier** (events, attendees, badges) appartiennent à l'organisation.
- Le **paiement** est une responsabilité d'entreprise, pas individuelle.
- Un user peut appartenir à plusieurs orgs avec des plans différents.
- Une org doit pouvoir **changer de billing owner** sans tout casser.
- Les **limites** (quotas) doivent être calculées par tenant, pas globalement par user.

---

## 2. Modèle conceptuel

```
Organization 1 ─── 1 Subscription ─── 1 Plan
     │                  │
     │                  └── billingOwnerUserId ──► User (membre de l'org)
     │
     └── memberships ──► Users
```

- **Organization** possède **une** Subscription active à un instant T.
- **Subscription** référence **un** Plan + porte le **billing owner** (user) + status + dates.
- **Plan** contient **features** et **limits**.
- **Billing owner** est un user **membre** de l'org (membership active), avec un rôle suffisant (Owner ou rôle dédié).

> Évolutions futures envisagées :
> - Historique de subscriptions (table `SubscriptionHistory` ou versioning).
> - Multiple subscriptions parallèles (add-ons) → écarté en V1.

---

## 3. Tables conceptuelles (sans migration)

Liste des entités à modéliser plus tard. **Aucune migration ni schéma Prisma ne doit être créée dans cette tâche.**

| Entité | Rôle | Notes |
|---|---|---|
| `User` | identité globale | déjà existant |
| `Organization` | tenant | déjà existant, à enrichir (`status`, `createdByUserId`) |
| `Membership` (`OrgUser`) | lien user ↔ org | déjà existant, à enrichir (`deletedAt` cf. doc user-deletion) |
| `Plan` | catalogue d'offres | nouveau — `code`, `name`, `features`, `limits`, `isPublic` |
| `Subscription` | offre active d'une org | nouveau — `orgId`, `planId`, `status`, `trialEndsAt`, `billingOwnerUserId` |
| `UsageCounter` *(optionnel)* | mesure courante (events, attendees) | nouveau — peut être calculé à la volée ou matérialisé |
| `PlanFeature` / `FeatureFlag` *(optionnel)* | activation fine | peut être JSON dans `Plan.features` au début |

> Démarrer **simple** : `Plan.features` et `Plan.limits` en JSON suffisent pour la V1. Tables séparées seulement si la complexité l'exige.

---

## 4. Exemple de plans

Indicatif, non figé. Les **limites** et **prix** seront tranchés en phase produit.

### Free

- 1 event actif
- 2 users (Owner + 1)
- Plafond attendees / event (ex. 100)
- Fonctionnalités de base : registration, check-in manuel
- Pas d'export avancé, pas d'intégration

### Starter

- Plusieurs events actifs (ex. 5)
- Plus de users (ex. 10)
- Exports CSV
- Badges PDF
- Branding minimal

### Pro

- Limites hautes (events / users / attendees)
- Fonctionnalités avancées : check-in mobile, partenaires, scanning
- Intégrations (n8n, webhooks, email custom)
- Support prioritaire

> Les prix ne sont **pas** documentés ici. Ils relèvent du produit/commercial.

---

## 5. Règle anti-abus Free

- Un user peut **créer plusieurs organisations**.
- Mais il ne peut avoir qu'**une seule organisation Free active créée par lui-même**.
- Les autres organisations qu'il crée doivent être en **Trial** ou en **plan payant**.
- Les **invitations ne comptent pas** comme création Free : être invité dans 10 orgs Free reste autorisé.

### Définition opérationnelle

- "Créée par lui" = `Organization.createdByUserId == user.id` (champ à ajouter conceptuellement).
- "Free active" = `Subscription.planCode == 'free'` ET `Subscription.status IN ('free', 'active')`.
- Quota = compter `COUNT(Organization WHERE createdByUserId=user AND subscription Free active) < 1` avant d'autoriser la création d'une nouvelle org en Free.

### Bypass possibles à NE PAS oublier

- Org **archived** ou **suspended** ne compte pas dans le quota.
- Org dont l'user a perdu l'ownership (transfert) : à arbitrer — recommandation : reste comptée tant que le user en est créateur initial, OU on suit l'ownership courant. À trancher.
- Platform Admin peut **outrepasser** la règle (création manuelle).

---

## 6. Vérification des limites

Toutes les limites doivent être vérifiées par rapport à **`currentOrgId`**, jamais par rapport à `userId`.

Points de vérification (non exhaustif) :

| Action | Limite | Source |
|---|---|---|
| Création event | `limits.maxEvents` | `Subscription` de `currentOrgId` |
| Invitation user | `limits.maxUsers` | idem |
| Ajout attendee | `limits.maxAttendeesPerEvent` | idem |
| Génération badge | `features.badges.enabled` | idem |
| Export CSV/PDF | `features.exports.enabled` | idem |
| Intégrations (n8n, webhooks) | `features.integrations[*]` | idem |
| Print queue | `features.print.enabled` | idem |

Pattern recommandé (conceptuel, **pas à implémenter ici**) :

```ts
// Pseudo
@CheckPlanLimit('events.create')
@Post('/events')
createEvent(...) { ... }
```

Décorateur ou guard qui résout le plan via `currentOrgId` et compare le compteur courant à la limite.

---

## 7. États possibles

### Organization status

- `active` — fonctionnement normal
- `suspended` — accès bloqué (impayé, abus, demande support) ; les memberships restent en base
- `archived` — org fermée volontairement ; lecture seule, pas de nouvelle action ; non comptée dans les quotas

### Subscription status

- `free` — plan Free permanent
- `trialing` — Trial en cours, `trialEndsAt` défini
- `active` — plan payant en règle
- `past_due` — paiement échoué, période de grâce
- `canceled` — annulée, fin de période en cours
- `suspended` — bloquée par Platform Admin (override)

### Transitions notables

- `trialing → active` : conversion (paiement réussi)
- `trialing → free` : fin de Trial sans conversion → downgrade automatique (si politique adoptée)
- `active → past_due → suspended` : escalade impayé
- `suspended → active` : réactivation manuelle

---

## 8. Points à auditer avant implémentation

À faire dans le ticket d'implémentation, **pas maintenant** :

- **Modèle Prisma existant** : `Organization`, `OrgUser`, présence/absence d'un champ `createdByUserId` sur `Organization`. Cf. [prisma/schema.prisma](../../attendee-ems-back/prisma/schema.prisma).
- **Champs Organization actuels** : status, slug, créateur, settings.
- **Logique `currentOrgId`** : où est-il défini, où est-il validé. Cf. [src/auth/interfaces/jwt-payload.interface.ts](../../attendee-ems-back/src/auth/interfaces/jwt-payload.interface.ts).
- **Invitations** : modèle existant, gestion du rôle proposé, expiration. Cf. `src/modules/invitations/`.
- **Users controller** : impacts sur soft-delete / hard-delete cf. [bugs/user-deletion-multitenant-membership.md](../bugs/a-faire/user-deletion-multitenant-membership.md).
- **Organization controller** : endpoints de création / update / list.
- **Permissions** : table `Permission` / `UserRole`, format `resource.action`.
- **Frontend org switcher** : gestion multi-org, refresh du JWT, persistance du choix.
- **Besoin Stripe ou non** : décision V1 vs V2 (probablement V2). Modélisation Subscription doit pouvoir accueillir un `stripeSubscriptionId` plus tard.
- **Besoin d'un usage counter** : calcul à la volée (COUNT SQL) suffisant en V1, matérialisation seulement si perf le justifie.
- **Règles de trial** : durée (14j / 30j ?), conversion automatique en Free ou suspension à l'expiration ?
- **Règle du dernier Owner** : interdire son retrait, forcer un transfert.
- **Impact sur les guards existants** : ajout d'un éventuel `CheckPlanLimitGuard`.
