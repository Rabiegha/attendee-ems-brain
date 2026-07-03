# SaaS Foundation — Organization Onboarding & Billing

> **Statut** : Documentation produit / Cible. **Aucun code modifié.**
> **Branche** : `feat/authz-m1-audit-service` (référence)
> **Public** : Produit, Architecture, Dev

---

## 1. Objectif

Clarifier la fondation SaaS d'Attendee pour rendre le produit simple à onboarder en multi-tenant :

- **Signup** sans friction (email + mot de passe).
- **Création d'un premier espace** (organisation) immédiate, sans étape de paiement.
- **Plan attaché à l'organisation**, pas au user (Free ou Trial par défaut).
- **Invitations** pour rejoindre d'autres espaces sans payer.
- Préparer le terrain pour un **billing futur** porté par l'organisation.
- Aligner UX, modèle de données et permissions sur le mental model **tenant-first**.

---

## 2. Modèle mental produit

- **User** = une **personne** (identité globale, email + password). Vit indépendamment des organisations.
- **Organization** = un **espace de travail** (entreprise, agence, équipe événementielle). C'est le tenant.
- **Membership** = l'**accès** d'un user à une organization (et son rôle tenant).
- **Plan / Subscription** = l'**offre active** d'une organization. Détermine features et limites. Appartient à l'organization, pas au user.

### Schéma texte

```
        ┌─────────┐                       ┌──────────────┐
        │  User   │◄──── Membership ─────►│ Organization │
        │ (perso) │      (role tenant)    │  (tenant)    │
        └─────────┘                       └──────┬───────┘
             ▲                                   │
             │                                   │ 1
             │                                   ▼
             │                            ┌──────────────┐
             │                            │ Subscription │──► Plan
             │                            └──────────────┘
             │                                   │
             └────── billingOwnerUserId ◄────────┘
                  (un user membre de l'org)
```

**Règles d'or** :
- Le plan/billing **n'appartient jamais** à un user.
- Un user **n'a pas** de "plan personnel". Seules ses organisations ont un plan.
- Les **données métier** (events, attendees, badges…) appartiennent à l'organisation.

---

## 3. Onboarding cible

### 3.1 Signup normal (premier user)

1. Création **User** (email, password, profil minimal).
2. Création **Organization** (nom + slug auto-généré, modifiable).
3. Création **Membership** avec rôle `Owner`.
4. Création **Subscription** par défaut (`Free` ou `Trial` selon décision produit — cf. roadmap §6).
5. Définition de `currentOrgId` dans le JWT pour la nouvelle org.
6. **Redirection directe** vers le dashboard de cette organisation.

### 3.2 Invitation (rejoindre une org existante)

1. Le user reçoit une **invitation** par email (lien tokenisé).
2. Il crée son compte **ou** se connecte à un compte existant.
3. Il accepte l'invitation → **Membership** créée dans l'org invitante avec le rôle proposé par l'inviteur.
4. **Pas** de création d'organisation, **pas** de création de Subscription.
5. `currentOrgId` = l'organisation invitante (ouverture directe sur l'espace rejoint).
6. Si le user appartient déjà à d'autres orgs, l'**org switcher** lui permet d'alterner.

### 3.3 Création d'une autre organisation (par un user existant)

1. Depuis l'org switcher ou le menu profil → **"Créer un nouvel espace"**.
2. Si le user a **déjà une org Free active créée par lui** :
   - blocage de la création en plan Free ;
   - propositions : **Trial** (durée à définir) ou **plan payant** (futur).
3. Création **Organization** + **Membership Owner** + **Subscription** (Trial ou plan choisi).
4. Switch automatique de `currentOrgId` vers la nouvelle org.

---

## 4. Règles produit

- Un user peut **créer plusieurs organisations**.
- Un user peut **rejoindre plusieurs organisations** via invitation.
- Chaque organisation a **son propre plan** (Subscription) indépendant.
- **Anti-abus Free** : un user ne peut avoir qu'**une seule organisation Free active** créée par lui-même.
- Les **invitations ne sont pas bloquées** par la règle Free : être invité dans 10 orgs Free reste autorisé (le quota se compte sur le créateur, pas sur le membre).
- Le **billing owner** d'une organisation est un user **membre** de l'org. Il peut être différent du créateur initial (transférable plus tard).
- Les **limites** (events, attendees, exports…) sont vérifiées sur la base de `currentOrgId`.

---

## 5. UX recommandée

Wording cible côté frontend (sans modification immédiate) :

| Au lieu de | Préférer |
|---|---|
| "Créer mon organisation" | **"Créer mon espace"** |
| "Organisation actuelle" | **"Espace actuel"** |
| "Plan de l'utilisateur" | **"Plan de cet espace"** |
| "Supprimer utilisateur" (vu depuis un admin org) | **"Retirer de l'organisation"** |
| "Désactiver cet utilisateur" (admin org) | **"Retirer de l'organisation"** |
| "Mon compte" (englobant tout) | Séparer : **"Mon profil"** (user global) vs **"Cet espace"** (org) |

Cf. [bugs/user-deletion-multitenant-membership.md](../bugs/a-faire/user-deletion-multitenant-membership.md) pour le wording côté suppression.

---

## 6. Cas limites

À gérer / documenter pour le futur ticket :

- **Nom d'organisation** déjà utilisé : autorisé (le slug fait foi).
- **Slug unique** : auto-généré, modifiable, contrainte d'unicité globale.
- **User invité qui possède déjà une org** : OK, ajoute juste une membership.
- **User qui quitte une org** : retrait de membership uniquement, son User reste actif (cf. doc bug user-deletion).
- **Dernier Owner / Admin** d'une org : interdire la sortie ou la suppression de membership. Transfert d'ownership requis.
- **Organisation suspendue** (impayé, abus) : empêcher login dans cette org, garder accès aux autres orgs du user.
- **User avec plusieurs orgs** : org switcher visible en permanence.
- **Changement de `currentOrgId`** : doit invalider/recharger le contexte permissions sans logout complet.
- **Création d'une seconde org Free** : refusée explicitement avec message "Vous avez déjà un espace Free — démarrez un Trial".
- **Membership soft-deleted** : ne doit plus apparaître dans le switcher.
- **JWT avec `currentOrgId` pointant vers une org supprimée/suspendue** : guard tenant rejette, force le switch.

---

## 7. Hors scope immédiat

Les sujets suivants sont **explicitement hors scope** de la fondation immédiate :

- Intégration **paiement réel Stripe** (modélisation seulement).
- **Facturation complète** (invoices, TVA, dunning).
- **Delete account / anonymisation RGPD** — chantier séparé (cf. [bugs/user-deletion-multitenant-membership.md](../bugs/a-faire/user-deletion-multitenant-membership.md) §7 cas C).
- Création d'un **nouveau frontend admin séparé** (`admin.attendee.fr`) — pour l'instant route `/platform` dans le front existant.
- **Marketplace** ou self-service avancé (downgrade auto, coupons publics).
- **SSO / SCIM** entreprise.
- **Multi-region / data residency**.
