# Chantier K — Résilience event

> Checklist finale : être sûr que **tout ce qui protège** est en place (saturation, disque,
> perte, recovery testé) avant J-7. Cadré dans le
> [workstream résilience event](../../../a-faire/resilience-event-lfd2026/README.md).

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (ligne K du tableau)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **K**)

## Raccord audit LFD

Avant le k6 final et la recette CDC, K vérifie avec [O](../O-audit-lfd/README.md) :

- alerte visible si une écriture d'audit obligatoire échoue ou si les échecs se répètent ;
- procédure pour diagnostiquer, corriger puis prouver la reprise sans fabriquer de faux événements ;
- contrôle du volume, de la croissance de `audit_logs` et du disque ;
- présence des événements obligatoires pendant le test combiné, sans secret dans les métadonnées.

K ne stocke pas les audits et ne construit pas de pipeline de logs : il porte la vérification
opérationnelle et le runbook de l'événement.

## Raccord scan terrain M

K organise la [recette terrain scan LFD](../M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md) :

- parc réel d'appareils et versions OS confirmé avec le client ;
- première répétition au plus tard J-10 et contre-recette au plus tard J-4 ;
- plusieurs scanners simultanés et réseaux online/offline/instables ;
- un observateur compare les téléphones au dashboard J-ENTREES : présents actuels, visiteurs uniques,
  capacité physique, taux et reprise correcte après reconnexion ;
- téléphones chargés, appareils de secours, chargeurs/batteries et connectivité de secours ;
- PV avec build, appareils, anomalies, owners et décision GO/NO-GO.

M corrige l'application et produit les preuves techniques ; K possède le rendez-vous, les moyens
terrain et la décision opérationnelle. Un smoke isolé ne remplace pas la répétition multi-appareils.

## Planning daté des répétitions scan

> Les dates sont calculées par rapport au premier jour de LFD, le **vendredi 4 septembre 2026**.
> Owner organisation : K. Owners corrections : M côté mobile, H/L9.1 côté serveur.

| Date | Jalon | Contenu | Sortie obligatoire |
| --- | --- | --- | --- |
| **Vendredi 21 août — J-14** | Préparation | build candidate, comptes scanner, sessions et QR de test, appareils et connectivités, observateurs M/H/K | checklist prête, aucun prérequis bloquant |
| **Mardi 25 août — J-10** | Répétition complète | plusieurs scanners, online/offline, même QR, dernière place, refus métier, reconnexion, queue à zéro, 15–30 min de scan continu ; contrôle simultané du dashboard salle/créneau | PV n°1, chiffres serveur/mobile/dashboard réconciliés, anomalies classées, owners et délais |
| **26–30 août — J-9 à J-5** | Corrections | corriger uniquement les anomalies bloquantes/majeures, rejouer automatisation H/M et produire un nouveau build si nécessaire | liste des blocants fermée ou décision NO-GO préparée |
| **Lundi 31 août — J-4** | Contre-recette | build final, reprise des scénarios échoués, smoke multi-appareils et export témoin avec un présent, un absent, un sorti et un walk-in | PV final et décision `GO`, `GO avec réserve` ou `NO-GO` |
| **Mercredi 2 septembre — J-2** | Gel/distribution | geler version/runtime, installer le build validé, connecter les comptes, synchroniser toutes les sessions, charger appareils et secours | parc prêt, versions et comptes inventoriés |
| **Jeudi 3 septembre — J-1** | Smoke opérationnelle | un QR valide, un refus, vérification offline/queue et dashboards ; aucune modification risquée | feu vert opérationnel final |
| **4–5 septembre — event** | Exploitation | dispositif 1+1 ou présence décidée, suivi queues/erreurs, escalade selon runbooks | journal d'incidents et décisions |

### Règles de calendrier

- réserver les créneaux J-10 et J-4 dès maintenant ;
- aucune répétition n'est validée sans build/commit, appareils, résultats et anomalies consignés ;
- une correction touchant caméra, auth, stockage, offline ou scan serveur après J-4 impose une
  contre-recette ciblée ;
- après le gel J-2, seuls les correctifs bloquants suivent le chemin hotfix et doivent être retestés ;
- si un bloqueur reste ouvert à J-4, K prononce `NO-GO` ou documente une solution de repli explicite.

## Fichiers du dossier

| Fichier                                                  | Sujet                                                                               |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| [tenir-event-sans-gcp.md](./tenir-event-sans-gcp.md)     | Décision — tenir l'event **sans** migration GCP (+ gate post-L9)                    |
| [dispositif-jour-j-devs.md](./dispositif-jour-j-devs.md) | DRAFT — présence des devs le jour J (1+1 recommandé vs 2 sur place avec conditions) |
| [recette scan M](../M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md) | Protocole appareils, offline, multi-scanner et GO/NO-GO |
