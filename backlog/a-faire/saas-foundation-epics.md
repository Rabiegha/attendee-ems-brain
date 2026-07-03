# SaaS Foundation — Epics & Phases

> **Statut** : Roadmap / Cadrage. **Aucun code modifié.**
> **Documents liés** :
> - [product/saas-foundation-onboarding-billing.md](../../../attendee-ems-back/docs/product/saas-foundation-onboarding-billing.md)
> - [architecture/platform-admin-access-model.md](../../../attendee-ems-back/docs/architecture/platform-admin-access-model.md)
> - [architecture/organization-plan-subscription-model.md](../../../attendee-ems-back/docs/architecture/organization-plan-subscription-model.md)
> - [bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md)
> - [bugs/cors-origin-security-review.md](../../bugs/a-faire/cors-origin-security-review.md)

---

## 1. Initiative

**SaaS Foundation — Onboarding, Plans & Platform Admin**

Initiative transverse qui consolide les fondations multi-tenant d'Attendee : onboarding clair, plans par organisation, console plateforme légère. Prépare le terrain au billing futur sans le livrer immédiatement.

---

## 2. Phase 1 — Organization Onboarding

### Objectifs

- Le signup crée systématiquement **User + Organization + Membership Owner** dans la même transaction.
- Slug d'organisation **unique** auto-généré + modifiable.
- `currentOrgId` défini cohéremment à la fin du signup et à chaque switch.
- **Flow d'invitation** robuste : un user invité rejoint l'org sans créer d'org.
- Wording frontend clarifié (espace / organisation / retirer).

### Tickets candidats

- [ ] **P1-T1** — Documenter le flow d'onboarding existant (état actuel) avant toute modification.
- [ ] **P1-T2** — Garantir la création atomique `User + Organization + Membership Owner` au signup.
- [ ] **P1-T3** — Gérer l'unicité du slug (génération + collision + édition).
- [ ] **P1-T4** — Créer/valider la Membership Owner à la création d'org.
- [ ] **P1-T5** — Définir et propager `currentOrgId` après signup (JWT + refresh).
- [ ] **P1-T6** — Vérifier et durcir le flow `accept invitation` (création membership uniquement, pas d'org).
- [ ] **P1-T7** — Refonte wording frontend : "espace", "Retirer de l'organisation", "Plan de cet espace".
- [ ] **P1-T8** — Tests e2e onboarding (signup, invitation, multi-org).

### Dépendances / risques

- Lié au bug user-deletion (cf. [bugs/user-deletion-multitenant-membership.md](../../bugs/a-faire/user-deletion-multitenant-membership.md)) — corriger d'abord la sémantique membership.
- Refactor de wording → impact i18n FR/EN.

---

## 3. Phase 2 — Free Rule & Basic Plan Visibility

### Objectifs

- Modèle **Plan** + **Subscription** minimal (Free + Trial, sans Stripe).
- Plan visible côté **Settings org** (lecture seule).
- Règle **"une seule org Free active créée par user"** appliquée à la création.
- Limites simples (events, users) vérifiées sur `currentOrgId`.

### Tickets candidats

- [ ] **P2-T1** — Documenter et valider le modèle `Plan` / `Subscription` (cf. doc architecture).
- [ ] **P2-T2** — Migration création tables `Plan`, `Subscription` (et `Organization.createdByUserId` si manquant).
- [ ] **P2-T3** — Seed plans `free`, `trial`, `starter`, `pro` (limits/features en JSON).
- [ ] **P2-T4** — Subscription Free auto-créée à chaque nouvelle org Phase 1.
- [ ] **P2-T5** — Vérification anti-abus : bloquer la création d'une 2e org Free par le même créateur.
- [ ] **P2-T6** — Proposer "Démarrer un Trial" en alternative.
- [ ] **P2-T7** — Endpoint `GET /organizations/:id/subscription` (lecture).
- [ ] **P2-T8** — UI Settings org : afficher plan courant + limites + usage.
- [ ] **P2-T9** — Guard / décorateur de vérification de limites pour `events.create` (premier cas pilote).

### Dépendances / risques

- Nécessite Phase 1 finalisée (membership owner fiable).
- Décision produit sur durée Trial + conversion → cf. §6.

---

## 4. Phase 3 — Platform Admin Console

### Objectifs

- Route **`/platform`** dans le frontend existant.
- Rôle **Platform Admin** séparé (table dédiée, indépendant des memberships).
- Liste des organisations + détail.
- Vue users globaux + memberships.
- Audit log minimal des actions plateforme.

### Tickets candidats

- [ ] **P3-T1** — Documenter et figer le modèle d'accès plateforme (cf. doc architecture platform-admin).
- [ ] **P3-T2** — Migration table `PlatformAccess` (userId, roles[]).
- [ ] **P3-T3** — `PlatformGuard` backend, séparé du `TenantGuard`.
- [ ] **P3-T4** — Routes `/api/platform/*` (orgs, users, subscriptions) en lecture.
- [ ] **P3-T5** — Layout frontend `/platform` (pas de sélecteur d'org tenant).
- [ ] **P3-T6** — Page liste organisations + filtres (status, plan).
- [ ] **P3-T7** — Page détail org : plan, status, membres, créateur, billing owner.
- [ ] **P3-T8** — Page users globaux + memberships d'un user.
- [ ] **P3-T9** — Actions minimales : changer plan, prolonger trial, suspendre/réactiver org.
- [ ] **P3-T10** — Audit log : capture de toutes les actions plateforme avec `actorUserId`.

### Hors scope Phase 3

- Impersonation (à valider séparément).
- Reset password / désactivation compte global (Cas B doc user-deletion) — Phase 4.

### Dépendances / risques

- Phase 2 fournit le modèle Subscription nécessaire.
- Sécurité : audit Platform Admin obligatoire avant mise en prod.

---

## 5. Phase 4 — Billing & Advanced Controls

### Objectifs

- Intégration **paiement** (Stripe ou équivalent) — à décider.
- Cycle de vie Subscription complet (`trialing → active → past_due → canceled`).
- **Usage counters** matérialisés si nécessaire.
- **Feature gating** fin par plan.
- **Suspension** automatisée org sur impayé.
- **Audit logs** étendus, exports SOC.
- Cas B user-deletion : désactivation compte global par Platform Admin.
- Cas C user-deletion : flow RGPD `delete my account`.

### Tickets candidats

- [ ] **P4-T1** — Design technique Stripe (ou alternative) + décision build vs buy.
- [ ] **P4-T2** — Webhooks Stripe → mise à jour Subscription.
- [ ] **P4-T3** — `BillingOwner` éditable (transfert).
- [ ] **P4-T4** — Invoices : modèle + endpoint + UI.
- [ ] **P4-T5** — Usage counters matérialisés (si perf le justifie).
- [ ] **P4-T6** — Feature gating étendu (badges, exports, intégrations).
- [ ] **P4-T7** — Subscription lifecycle automatisé (trial expiration, dunning).
- [ ] **P4-T8** — Suspension auto org sur `past_due` > N jours.
- [ ] **P4-T9** — Cas B : désactivation globale compte (Platform Admin).
- [ ] **P4-T10** — Cas C : flow `delete my account` (RGPD) avec anonymisation.
- [ ] **P4-T11** — Impersonation contrôlée Platform Admin (si validée produit + sécu + juridique).

### Dépendances / risques

- Décisions juridiques (RGPD, CGU, facturation) bloquantes pour Cas C.
- Stripe = engagement long terme : ne pas se précipiter.

---

## 6. Décisions ouvertes

À trancher avant ou pendant Phase 2/3 :

- **Plan par défaut au signup** : Free direct ? Trial 14j puis bascule Free ? Trial 30j puis suspension ?
- **Durée du Trial** : 7j / 14j / 30j ?
- **Limites Free précises** : combien d'events, users, attendees ?
- **Deuxième org Trial autorisée** ? Si oui, combien de Trials parallèles par user ?
- **Qui peut être billing owner** : Owner uniquement ? Owner + Admin ? Rôle dédié ?
- **Dernier Owner** : interdiction stricte de suppression de membership / transfert obligatoire ?
- **Route `/platform` vs `admin.attendee.fr`** : maintenant `/platform`, mais à quel seuil migrer ? (nombre de fonctionnalités, équipe ?)
- **Stripe maintenant ou plus tard** : recommandation Phase 4, mais si un client demande une facture custom en attendant ?
- **Feature flags** : utiliser un outil externe (LaunchDarkly, Unleash) ou stocker dans `Plan.features` JSON ?
- **Impersonation** : autorisée ou non ? Si oui, write actions bloquées ou tracées ?
- **Audit log** : table interne, ou outil externe (Datadog, Logtail) ?

---

## 7. Recommandation de priorité

Ordre conseillé pour limiter le risque et livrer de la valeur progressivement :

1. **Documenter l'existant** (cette doc + bugs adjacents) ✅ en cours.
2. **Sécuriser l'onboarding org** (Phase 1) — préalable à tout le reste.
3. **Clarifier membership & `currentOrgId`** — corriger d'abord le bug [user-deletion-multitenant-membership](../../bugs/a-faire/user-deletion-multitenant-membership.md) avant d'ajouter de nouveaux flows.
4. **Ajouter visibilité plan simple** (Phase 2 light : modèle + lecture seule + règle Free) — sans Stripe.
5. **Platform Admin minimal** (Phase 3 : liste orgs + détail + actions support basiques).
6. **Billing avancé + cas user-deletion B/C** (Phase 4) — uniquement quand les fondations sont stables.

### Anti-pattern à éviter

- ❌ Démarrer par Stripe avant d'avoir clarifié le modèle Plan/Subscription.
- ❌ Démarrer par un nouveau frontend admin séparé.
- ❌ Mélanger rôles tenant et rôles plateforme dans la même table.
- ❌ Implémenter l'impersonation sans validation produit + sécurité + juridique.
- ❌ Corriger la suppression user dans le même ticket que le modèle de plan (deux chantiers distincts).
