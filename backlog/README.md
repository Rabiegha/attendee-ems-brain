# Backlog

Sujets pour **plus tard**, toutes apps confondues. Vient de l'inbox `[idea]`.

> **Documenter ≠ faire.** Ne rien implémenter d'ici sans le passer d'abord par un `NOW.md` (workspace).

## Découpe (3 états)

| Sous-dossier           | Contient                                                  |
| ---------------------- | --------------------------------------------------------- |
| [fait/](fait/)         | Sujets backlog terminés (archive courte avant nettoyage). |
| [en-cours/](en-cours/) | Sujets démarrés mais pas finis.                           |
| [a-faire/](a-faire/)   | Sujets pas encore commencés.                              |

## En cours — index

| Sujet                                                                  | App           | Note                                                                                                 |
| ---------------------------------------------------------------------- | ------------- | ---------------------------------------------------------------------------------------------------- |
| [Appli mobile — préparation LFD2026](en-cours/appli-mobile-lfd2026.md) | mobile        | Mot de passe oublié, pagination sessions, offline scans, use cases scan, version stable pour l'event |
| [Sujets parkés — hors-scope](en-cours/sujets-parkes-hors-scope.md)     | mobile + back | Refonte nav mobile, migration persistance MMKV, sécurité EAS, async architecture (aiguillage).       |

## À faire — index

| Sujet                                                                                                           | App           | Note                                                                                                                                                |
| --------------------------------------------------------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Backlog technique](a-faire/BACKLOG-TECH.md)                                                                    | transversal   | Chantiers techniques (faux succès scan session, idempotence locale, OTA, print offline…). Contient lui-même ses sections Fait / En cours / À faire. |
| [Plan CI/CD](../workstreams/en-cours/lfd2026/0-fondations/CI_CD_PLAN.md)                                        | infra         | Mise en place de l'intégration / déploiement continu.                                                                                               |
| [Backup automatique DB](fait/brief-backup-automatique-db.md)                                                    | back/infra    | Cron de dump + rétention sur le VPS.                                                                                                                |
| [Garde-fous déploiement staging](../workstreams/en-cours/lfd2026/A-I-leviers/garde-fous-deploiement-staging.md) | infra         | 6 garde-fous anti-collision prod (post-mortem 502 du 25/06 + résurrection branche du 09/07) — tableau fait/à faire.                                 |
| [Onboarding multi-tenant](a-faire/onboarding-multi-tenant.md)                                                   | back          | Création d'organization, premier user, invitations, membership.                                                                                     |
| [Billing & plan management](a-faire/billing-plan-management.md)                                                 | back          | Visibilité plan / subscription tenant, compteurs de limites, base Stripe (lecture seule).                                                           |
| [Platform Admin Console](a-faire/platform-admin-console.md)                                                     | back          | Console plateforme éditeur (section `/platform` MVP).                                                                                               |
| [Public landing page namespaces](a-faire/public-landing-page-namespaces.md)                                     | back          | Slug public d'org séparé du slug interne, réservation slugs premium.                                                                                |
| [Context mesh improvements](a-faire/context-mesh-improvements.md)                                               | transversal   | Structuration du context (règles IA + dev) à travers les repos.                                                                                     |
| [Refactoring & cleanup](a-faire/refactoring-cleanup.md)                                                         | back          | Liste cible des refactorings et nettoyages, au compte-gouttes.                                                                                      |
| [QR code stability & registration identity](a-faire/qr-code-stability-and-registration-identity/README.md)      | back + mobile | Stabilité QR, réconciliation identité d'inscription, modèle de credential.                                                                          |
| [SaaS Foundation — Epics & Phases](a-faire/saas-foundation-epics.md)                                            | back + front  | Roadmap SaaS découpée en phases (onboarding, plans, platform admin).                                                                                |
| [Archive migrée — `other/`](a-faire/other/)                                                                     | back / infra  | Notes & plans détaillés migrés depuis `back/docs/other/` (voir découpe ci-dessous).                                                                 |

## `other/` — archive migrée (détail)

Documents de fond migrés depuis `attendee-ems-back/docs/other/`. Matière brute conservée pour référence, pas des tâches priorisées.

| Sous-dossier                                                                                   | Contient                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [ARCH](a-faire/other/migrations%20and%20refactoring/ARCH/)                                     | Diagnostic archi hexagonale, guide Redis/BullMQ event-driven, archi event-driven.                                                                                                                                           |
| [Refactore-autorization](a-faire/other/migrations%20and%20refactoring/Refactore-autorization/) | Analyse permissions, passport/JWT, guides AUTHZ, review V1, refonte V2 + [Plan](a-faire/other/migrations%20and%20refactoring/Refactore-autorization/Plan/00-INDEX.md) (paliers, refactor front/mobile, tests, déploiement). |
| [migration-GCP](a-faire/other/migrations%20and%20refactoring/migration-GCP/)                   | Analyse capacité infra, audit auth Cloud Run, migration organisation GitHub, refactoring event-driven (FR/EN).                                                                                                              |
| [seeders](a-faire/other/seeders/SEED_EVENT_GUIDE.md)                                           | Guide de seed d'événement.                                                                                                                                                                                                  |
