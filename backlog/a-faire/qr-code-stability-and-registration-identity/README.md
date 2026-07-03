# QR Code Stability & Registration Identity Reconciliation

> **STATUS : TO REVIEW / DISCOVERY**
> **NOT READY FOR IMPLEMENTATION**
>
> Backlog de découverte uniquement.
> Aucune migration, aucun changement de modèle, aucun code à produire à ce stade.

---

## 1. Objectif du backlog

Analyser un risque produit/technique critique : **la stabilité des QR codes envoyés aux invités d'un événement**, en particulier lorsque la donnée sous-jacente (`registration`, `attendee`) peut être modifiée, supprimée ou réimportée entre le moment où le QR est généré/envoyé et le jour J.

Ce backlog a pour but de :

- formaliser le problème ;
- documenter les cas d'usage à couvrir ;
- évaluer plusieurs options de solution ;
- proposer un modèle conceptuel cible (QrCredential + stableIdentityKey) ;
- préparer un futur plan d'implémentation par phases ;
- **rien implémenter pour l'instant**.

## 2. Pourquoi ce sujet est critique

Le QR code est l'un des artefacts les plus exposés du produit :

- il quitte la plateforme dès qu'un email est envoyé ou qu'un badge est imprimé ;
- une fois sorti, il devient une **promesse contractuelle** envers l'invité ;
- son échec se manifeste au pire moment : à l'entrée de l'événement, devant l'invité, devant l'organisateur, devant les hôtesses ;
- la cause racine est souvent invisible côté utilisateur (suppression/réimport d'une liste, refonte d'attendees côté admin) ;
- l'impact image, support et confiance est disproportionné par rapport à la cause technique.

Pour un SaaS multi-tenant événementiel, c'est un risque produit majeur.

## 3. Résumé du problème

Aujourd'hui, le QR code semble encoder directement un **identifiant technique** (probablement `registration.id`, voir [03-current-risk-analysis.md](./03-current-risk-analysis.md)).

Ce choix est fragile parce que :

- la donnée peut être recréée avec un nouvel ID après suppression/réimport ;
- un QR déjà envoyé devient alors silencieusement orphelin ;
- aucun mécanisme de réconciliation ne permet de retrouver « la même personne » dans le nouvel état de la base.

Le vrai problème n'est donc pas « comment changer le QR code », mais :

> **Comment garantir qu'un QR déjà envoyé reste résolvable vers la bonne inscription, malgré :**
>
> - une modification de données (nom, entreprise, type de participant) ;
> - un réimport de la liste avec le même email/external reference ;
> - une suppression puis recréation technique de l'inscription ;
> - un changement de `registration.id` ;
> - un changement de `attendee.id`.

## 4. Résumé de l'idée cible

L'orientation provisoire (à challenger dans [04-solution-options.md](./04-solution-options.md)) est de combiner :

1. **Ne plus exposer un ID technique** dans le QR public.
2. **Introduire un `QrCredential`** : entité dédiée au cycle de vie du QR (token, statut, révocation, remplacement, historique).
3. **Introduire une `stableIdentityKey`** sur la registration : une clé d'identité qui survit aux réimports (basée sur `external_reference`, `employee_id`, email normalisé, etc.).
4. **Préserver les QR existants au réimport** : la réconciliation d'identité doit retrouver la registration plutôt que d'en créer une nouvelle.
5. **Encadrer le hard delete** : interdire ou exiger une confirmation explicite si un QR a déjà été envoyé/imprimé/scanné.
6. **Résolution de fallback** : si l'ancien `registrationId` n'existe plus, le scan tente une résolution via `eventId + stableIdentityKey`.

## 5. In scope

- Conception du modèle d'identité stable (`stableIdentityKey`).
- Conception conceptuelle d'un modèle `QrCredential` (sans Prisma, sans migration).
- Définition du comportement de scan cible.
- Définition des règles de réconciliation à l'import.
- Définition des règles de suppression sécurisée.
- Liste exhaustive des cas d'usage à supporter.
- Liste des choses à ne pas faire.
- Critères d'acceptation préliminaires.
- Plan d'implémentation futur en phases (sans exécution).

## 6. Out of scope

- Implémentation effective (services, contrôleurs, DTOs).
- Migration Prisma.
- Modification du schéma `prisma/schema.prisma`.
- Modification de l'API mobile/front existante.
- Refonte du système d'import existant.
- Discussion du composant de scan UI (caméra, ML, etc.).
- Choix d'un format de token (JWT vs random) — sera tranché en Phase 1.
- Sécurité cryptographique avancée (signature, anti-cloning physique du QR) — voir Option E uniquement à titre comparatif.

## 7. Ordre de lecture recommandé

1. [01-problem-statement.md](./01-problem-statement.md) — Le problème en clair.
2. [02-use-cases.md](./02-use-cases.md) — Ce qu'on doit savoir gérer.
3. [03-current-risk-analysis.md](./03-current-risk-analysis.md) — Ce qui existe aujourd'hui (sourcé dans le code).
4. [04-solution-options.md](./04-solution-options.md) — Les options comparées.
5. [05-registration-identity-reconciliation.md](./05-registration-identity-reconciliation.md) — Le concept central de `stableIdentityKey`.
6. [06-qr-credential-model.md](./06-qr-credential-model.md) — Le modèle conceptuel cible.
7. [07-scan-behavior.md](./07-scan-behavior.md) — Le comportement de scan attendu.
8. [08-things-to-avoid.md](./08-things-to-avoid.md) — Les anti-patterns à proscrire.
9. [09-open-questions.md](./09-open-questions.md) — Les questions à trancher avec le produit.
10. [10-acceptance-criteria-draft.md](./10-acceptance-criteria-draft.md) — Critères d'acceptation provisoires.
11. [11-future-implementation-plan-draft.md](./11-future-implementation-plan-draft.md) — Plan futur par phases.

---

## Tags

`backlog` `discovery` `qr-code` `registration` `identity` `import` `reconciliation` `check-in` `risk:high` `status:to-review`
