# Learning — restauration du suivi chantiers LFD

Date : 2026-07-17

## Pourquoi cette note existe

Le fichier `03-suivi-chantiers.md` donnait l'impression d'être revenu à une version ancienne :

- `0-MON` était encore affiché à 90 %, alors que le monitoring prod était considéré opérationnel.
- `BIL` était encore affiché à 40 % et marqué hors estimation.
- `B0` était encore affiché à 40 %, alors que le déploiement privé Gotenberg/Cloud Run avait avancé.
- Le rapport W29 reprenait les mêmes valeurs anciennes.

La cause probable est une divergence de branches docs : la branche `agent/finalize-monitoring-docs` contenait une version plus complète du suivi, puis `main` a continué avec les ajouts L9b/C2.1/C3 sans reprendre tous les avancements de cette branche.

## Correction faite

Le suivi actuel a été réconcilié sans supprimer les ajouts récents :

- conservation de L3/L9b sur staging, PR #35, C2.1 et C3 warm-up ;
- restauration de `0-MON` à 100 % avec détail prod opérationnel ;
- restauration de `B0` à 70 % avec Gotenberg/Cloud Run privé ;
- réintégration de `BIL` dans l'estimation globale, puis correction à 55 % pour ne pas sur-vendre le chantier ;
- alignement du rapport hebdo W29 avec ces valeurs.

Après audit, d'autres fichiers utiles présents sur `agent/finalize-monitoring-docs` mais absents
de `main` ont aussi été restaurés :

- `0-fondations/0-MON-recap-monitoring-actif.md` ;
- `B-email-billet-pdf/suivi-gcp-gotenberg-auth-cloudrun.md` ;
- `C-migration-esp/C2-1-email-billet-session/00-suivi.md` ;
- `C-migration-esp/C2-1-email-billet-session/01-etat-des-lieux.md` ;
- `C-migration-esp/C2-1-email-billet-session/02-plan.md` ;
- `C-migration-esp/C2-1-email-billet-session/README.md`.

Les index/README ont été réalignés pour pointer vers ces fichiers.

## Décision sur BIL 55 %

La valeur retenue devient **55 %** pour le chantier global :

- billetterie publique one-page ;
- agenda 2 jours ;
- back-office client ;
- synchronisation Attendee ;
- inscription publique session ;
- jauges live.

Le socle MVP LFD est avancé, mais ce n'est pas une plateforme billetterie générique post-event
avec offres/paiement, builder multi-event avancé et self-service complet. Cette nuance doit rester visible dans
`00-plan-action.md`, `03-suivi-chantiers.md` et `BIL-billetterie/README.md`.

## Règle à retenir

Quand plusieurs branches docs existent, ne pas reprendre un fichier complet depuis une branche ancienne. Il faut faire une réconciliation ligne par ligne :

1. garder les ajouts les plus récents de `main` ;
2. récupérer les chantiers mieux renseignés dans les branches docs ;
3. vérifier les pourcentages dans `03-suivi-chantiers.md` et dans le rapport hebdo ;
4. vérifier les README/index des dossiers concernés ;
5. ne pas modifier les fichiers dirty non liés.

## État cible après correction

- `0-MON` : 100 %, livré/opérationnel.
- `B0` : 70 %, Cloud Run Gotenberg privé prêt côté GCP, branchement back restant.
- `BIL` : 55 %, vrai chantier produit inclus dans l'estimation, socle avancé mais finalisation restante.
- `C2.1` : 20 %, cadrage fait, implémentation restante côté Attendee/BullMQ.
