# Rapports de load tests — Session autonome LFD 2026

Ce dossier reçoit un rapport Markdown pour **chaque** exécution de charge, y compris un test échoué,
interrompu ou invalidé par un problème de générateur. Un bilan final compare tous les scénarios et
formule un avis explicite sur la préparation de LFD.

## Convention de nommage

`YYYY-MM-DD-HHMM-<levier>-<scenario>.md`

Exemples :

- `2026-07-21-2215-l9b-baseline-50vu.md` ;
- `2026-07-21-2300-l9b-arrival-50rps.md` ;
- `2026-07-22-0015-l9a-burst-3000-60s.md`.

Les exports bruts volumineux ne doivent pas être commités sans vérifier leur taille et l'absence de
données personnelles. Le rapport versionné contient les agrégats utiles et référence l'emplacement
sécurisé des artefacts éventuels.

## Modèle obligatoire par exécution

```md
# Load test — <levier> — <scénario>

## Verdict rapide

- Statut : VALIDE / INVALIDE / INTERROMPU
- Niveau LFD : VERT / ORANGE / ROUGE
- Conclusion en une phrase :

## Identité du test

- Date et heure Europe/Paris :
- Auteur/exécutant :
- Environnement : staging uniquement
- URL/API ciblée, sans secret :
- Branche et commit du script :
- SHA backend réellement déployé :
- Version k6 :
- Endpoint et levier :

## Objectif et hypothèse

- Ce que le test cherche à prouver :
- Résultat avant changement servant de comparaison :
- Seuils techniques appliqués :
- Ce que ce test ne permet pas de conclure :

## Données et préconditions

- Événement/session synthétique :
- Capacité configurée :
- Nombre de comptes/identités uniques :
- État initial des compteurs :
- Méthode de préparation et de nettoyage :
- Vérification explicite que la cible n'est pas la production :

## Profil de charge

- Exécuteur k6 :
- VUs préalloués/max :
- Débit visé et débit réellement injecté :
- Ramp-up, durée de palier, burst et recovery :
- Nombre total d'itérations/requêtes :
- Générateur unique ou distribué :

## Santé du ou des générateurs k6

- CPU maximum par générateur :
- RAM physique maximale et swap :
- Bande passante maximale :
- Limite de fichiers/sockets et erreurs locales :
- `dropped_iterations` :
- Débit configuré comparé au débit réellement démarré :
- Répartition de charge par générateur :
- Le générateur peut-il être exclu comme goulot : oui/non, preuve :

## Résultats k6

| Mesure | Résultat | Seuil | Verdict |
| --- | ---: | ---: | --- |
| Tentatives métier | | | |
| Résultats métier attendus | | | |
| Débit injecté (req/s) | | | |
| Débit métier abouti (op/s) | | | |
| Échecs HTTP inattendus | | | |
| `5xx` inexpliqués | | 0 | |
| Médiane | | | |
| Moyenne | | | |
| p95 | | | |
| p99 | | | |
| Maximum | | | |
| Durée d'absorption du burst | | | |

## Invariants métier après test

- Surbooking : oui/non, preuve :
- Dérive du compteur : oui/non, preuve :
- Doublons : oui/non, preuve :
- Idempotence : conforme/non conforme, preuve :
- Répartition confirmed/waitlisted/refused :
- Cohérence DB/API/dashboard :

## Santé de l'architecture

- CPU/RAM API :
- Latence et saturation PostgreSQL :
- Pool/PgBouncer :
- Redis/BullMQ si concernés :
- Timeouts et files d'attente :
- Redémarrages, OOM ou erreurs réseau :
- Temps de retour à l'état normal :
- Limite observée : application, DB, réseau ou générateur k6 :

## Comparaison

- Différence avec la baseline :
- Différence avec le palier précédent :
- Gain ou régression p95/p99 :
- Gain ou régression de débit :
- Changement de consommation des ressources :

## Anomalies et corrections

- Symptômes :
- Cause confirmée ou hypothèses :
- Correction appliquée :
- Tests relancés après correction :
- Risque résiduel :

## Avis pour LFD

- Ce que le résultat prouve :
- Ce qu'il ne prouve pas :
- Niveau de confiance : faible/moyen/fort
- Risque pendant l'ouverture des inscriptions :
- Décision recommandée : poursuivre / corriger / changer l'architecture / GO technique
- Prochain test nécessaire :

## Nettoyage et traçabilité

- Données synthétiques nettoyées : oui/non
- Données conservées volontairement et raison :
- Lien PR/CI/CD :
- Emplacement des artefacts sans données personnelles :
```

## Grille d'avis LFD

### VERT

- aucun invariant métier violé ;
- aucun `5xx` inexpliqué ;
- débit visé réellement injecté et absorbé ;
- latences et ressources stables ;
- récupération normale après le burst ;
- résultat reproductible.

### ORANGE

- intégrité métier correcte, mais latence, saturation, récupération ou reproductibilité insuffisante ;
- le risque et la limite mesurée sont identifiés ;
- une correction ou un test complémentaire est requis avant le GO technique.

### ROUGE

- surbooking, compteur incorrect, doublon non maîtrisé ou perte de donnée ;
- `5xx` inexpliqués ou indisponibilité ;
- débit réellement injecté très inférieur au débit annoncé ;
- saturation sans récupération fiable.

Un test est **INVALIDE**, et non ROUGE pour Attendee, si le générateur atteint 80 % CPU, 90 % de la
RAM physique, swappe, sature son réseau ou ne démarre pas la charge configurée alors qu'Attendee reste
sous-utilisé. Il doit être optimisé ou distribué puis rejoué.

Un résultat VERT représente un **GO technique pour le scénario testé**, pas automatiquement une
preuve de conformité contractuelle aux « 3 000 simultanés ». Cette dernière exige que le client ou le
contrat définisse précisément la fenêtre d'arrivée et les seuils de latence acceptés.

## Bilan final de campagne

Le bilan final doit présenter :

- un tableau de tous les tests, SHA, charges, p95/p99, débit, erreurs et verdicts ;
- la courbe ou la description du point de saturation ;
- le meilleur débit stable et reproductible ;
- les performances avant/après chaque levier ;
- les limites de staging susceptibles de différer de la production ;
- les risques LFD restants classés par gravité ;
- une recommandation finale argumentée : GO, GO sous conditions ou NO-GO ;
- la liste exacte des actions à réaliser avant l'ouverture des inscriptions.
