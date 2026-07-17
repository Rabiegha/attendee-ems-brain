# Chantiers 0 — Fondations minimales (CI/CD + monitoring)

> **Regroupe :** **0-CI** (CI/CD minimaliste : build+test PR + deploy staging) et **0-MON**
> (monitoring & alerting minimal : uptime + Sentry + CPU + 🔴 alerte disque). Deux chantiers
> **transverses** qui protègent tous les autres — à poser tôt.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§P0 — Fondations minimales)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (lignes **0-CI** et **0-MON**)
- **Owner :** Rabie — **Statut :** CI/CD quasi fini, **0-MON prod opérationnel** (Netdata/Sentry/alertes/timer BullMQ)

## Fichiers du dossier

| Fichier                                                                    | Sujet                                                                                                                   |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| [finalisation-ci-cd-et-livraison.md](./finalisation-ci-cd-et-livraison.md) | 🎯 **Checklist d'exécution VIVANTE** — prérequis ops, validation pipeline, alignement branches, nettoyage, verrouillage |
| [0-CI-plan.md](./0-CI-plan.md)                                             | **Plan CI/CD ACTIF** — staging + prod (prod prioritaire), arbitrages actés, ordre de livraison                          |
| [CI_CD_PLAN.md](./CI_CD_PLAN.md)                                           | Annexe — squelette détaillé (jobs CI, tests E2E, fichiers à créer, plan de vérif)                                       |
| [0-MON-plan.md](./0-MON-plan.md)                                           | **Plan Monitoring ACTIF** — Sentry (déjà câblé) + uptime (fait) + Netdata/alertes disque à poser                        |
| [0-MON-recap-monitoring-actif.md](./0-MON-recap-monitoring-actif.md)       | **Recap 0-MON actif** — état prod, alertes, timer BullMQ, points hors MON                                                |
| [ci-cd-post-event.md](./ci-cd-post-event.md)                               | 📦 **Post-event** — gestion des tests en CI à l'échelle (étagement, parallélisation, déclencheurs)                      |
