# Rabbit Hole — Separate admin frontend now

Type: Rabbit Hole
Status: Watchlist
Risk level: Medium-High

## 1. Why it is tempting

- Séparer la console Platform Admin du frontend tenant semble « plus propre ».
- Permettrait des permissions / auth différentes.
- Évite que des composants admin polluent le bundle tenant.
- Patterns SaaS classiques (app principale + app admin séparée).

## 2. Why it is dangerous now

- Maintenir **deux frontends** = double CI/CD, double déploiement, double bundle, double auth, double session, double design system.
- Aucun Platform Admin opérationnel aujourd'hui → on créerait une app vide.
- L'équipe est petite, ajouter un repo / projet front double la charge ops.
- Le besoin réel pour le MVP est très limité (peut-être 3-5 écrans).
- Décision déjà actée : pour le MVP, préférer une **section `/platform`** dans le frontend existant (voir [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#platform-admin)).

## 3. Scope creep risk

Très élevé :
- Une fois la deuxième app créée, il faut y mettre auth, design, navigation, layout, error tracking, build pipeline.
- Risque d'y déverser des fonctionnalités « parce que c'est l'app admin » alors qu'elles n'existent pas encore.
- Risque de divergence : un composant tenant deviendrait incompatible avec l'admin et vice versa.
- Coût d'hébergement supplémentaire (sous-domaine, TLS, build, déploiement).

## 4. Current decision

- **Pas de second frontend pour le MVP.**
- La console admin se fera **à l'intérieur** du frontend tenant existant via une section `/platform` protégée par un Guard `SUPER_ADMIN`.
- Référence : [../architecture/platform-admin-access-model.md](../../architecture/platform-admin-access-model.md).

## 5. When to revisit

- Quand le périmètre admin dépassera **clairement** ce qu'une simple section interne peut absorber (≥ 20 écrans, équipe admin dédiée, designs très différents).
- Quand un besoin de sécurité forte (ex : IP allowlist différente) sera incontournable.
- Pas avant le MVP Platform Admin terminé.

## 6. Related docs

- [../backlog/platform-admin-console.md](../../backlog/a-faire/platform-admin-console.md)
- [../architecture/platform-admin-access-model.md](../../architecture/platform-admin-access-model.md)
- [../steering/DECISIONS.md](../../../attendee-ems-back/docs/steering/DECISIONS.md#platform-admin)
- [../workstreams/onboarding-billing-management/README.md](../../workstreams/a-faire/async-architecture/README.md)

## 7. Rule for Copilot

- Si une proposition contient « créer un second frontend admin », « nouvelle app `attendee-admin-front` », « repo dédié pour la console » → **stop**, pointer vers ce document.
- Autorisé : préparer une section `/platform` dans le front existant **avec ticket explicite**.
- Interdit : créer un nouveau workspace front, copier un boilerplate Vite/Next dédié, installer un nouveau déploiement.
