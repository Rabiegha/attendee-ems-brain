# Rabbit Hole — Full docs reorganization

Type: Rabbit Hole
Status: Watchlist
Risk level: Medium

## 1. Why it is tempting

- `docs/` contient plusieurs dossiers historiques (`ARCH/`, `Refactore-autorization/`, `archive/`, `migration-GCP/`, `seeders/`, `async-architecture/`).
- Certains fichiers semblent dupliqués ou mal classés.
- La nouvelle structure (`steering/`, `workstreams/`, `backlog/`, `rabbit-holes/`) appelle naturellement à « tout aligner ».
- Copilot peut être tenté de déplacer / fusionner / renommer pour « rendre cohérent ».
- Plusieurs repos (`attendee-context-hub/`, `ems-brain/`) ajoutent à la confusion → tentation de tout fusionner.

## 2. Why it is dangerous now

- Déplacer un fichier markdown **casse les liens** dans tous les autres fichiers, dans les PRs récentes, et dans la mémoire des agents.
- Renommer un dossier (`async-architecture/` → `workstreams/async-architecture/`) casse les `git log` / les bookmarks / les liens externes.
- Une réorg massive **noie le diff** : les vraies décisions structurantes deviennent illisibles.
- Le contexte mesh cross-repos est **fragile** ; une réorg côté back peut désaligner front/mobile/print/public-form/context-hub.
- Aucun bénéfice utilisateur ; pur confort interne.

## 3. Scope creep risk

Élevé :
- « Réorganiser docs » dérive vers « réorganiser context/ » de chaque repo.
- « Renommer un dossier » dérive vers « fixer tous les liens cassés » dans toute la doc.
- « Fusionner ARCH et architecture » dérive vers une revue éditoriale complète.
- « Aligner attendee-context-hub » dérive vers le chantier Context Mesh complet ([../backlog/context-mesh-improvements.md](../backlog/context-mesh-improvements.md)).

## 4. Current decision

- **Pas de réorg massive.**
- La structure cible (`steering/`, `workstreams/`, `backlog/`, `rabbit-holes/`) est **additive** : on **crée** ces dossiers, on **ne supprime pas** l'existant, on **ne déplace pas** sauf si l'appartenance est évidente et sans risque.
- Les dossiers historiques (`ARCH/`, `Refactore-autorization/`, `archive/`, `migration-GCP/`, `seeders/`, `async-architecture/`) restent **conservés et référencés**.
- Toute idée de déplacement → fiche dans [../backlog/context-mesh-improvements.md](../backlog/context-mesh-improvements.md), pas exécution directe.

## 5. When to revisit

- Quand le workstream **Context Mesh** sera lancé formellement.
- Quand les conventions cross-repo seront actées.
- Jamais comme initiative isolée.

## 6. Related docs

- [../backlog/context-mesh-improvements.md](../backlog/context-mesh-improvements.md)
- [../00-CONTROL-CENTER.md](../00-CONTROL-CENTER.md) (section 8 : Key folders).
- [../steering/README.md](../steering/README.md)

## 7. Rule for Copilot

- Si une proposition contient « réorganiser docs », « fusionner ARCH et architecture », « déplacer tous les fichiers de `async-architecture/` », « renommer le dossier `Refactore-autorization/` » → **stop**, pointer vers ce document.
- Autorisé : **créer** de nouveaux dossiers / fichiers (additif).
- Autorisé : **référencer** des fichiers existants depuis les nouveaux dossiers (liens markdown).
- Interdit : **supprimer** un dossier ou un fichier existant.
- Interdit : **déplacer** un markdown dont l'appartenance n'est pas évidente (en cas de doute, référence-le, ne le bouge pas).
- Interdit : **renommer** un dossier historique.
