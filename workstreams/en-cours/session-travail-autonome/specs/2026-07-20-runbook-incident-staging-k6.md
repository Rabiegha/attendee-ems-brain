# Runbook incident staging pendant une charge k6 autonome

> **Date de vérification :** 2026-07-20
>
> **Cible :** staging uniquement
>
> **Principe :** arrêter la pression, protéger les données et la production, puis récupérer uniquement
> les composants staging explicitement autorisés

## 1. État vérifié avant session

- `https://staging.attendee.fr/api/health` répond `200` avec DB, Redis et migrations à `ok` ;
- la version exposée par le healthcheck correspond au SHA courant de `origin/staging` ;
- l'accès SSH au VPS fonctionne depuis le poste audité ;
- l'utilisateur SSH dispose de Docker et de `sudo` non interactif ;
- `ems-staging-api` existe, est sain et possède `restart: unless-stopped` ;
- le workflow GitHub Actions `CD staging` est actif et ses secrets SSH sont configurés ;
- le CD staging possède un healthcheck et un rollback automatique de déploiement ;
- k6 n'est pas encore installé sur le poste audité, mais Homebrew est disponible ;
- aucun runner k6 actif et validé n'existe actuellement sur `staging` ; les anciens scripts sont dans
  `staging-archive-2026-07-10` et certains désactivent explicitement `abortOnFail` ;
- le watchdog exécutable décrit dans ce runbook reste donc à construire et tester avant la charge ;
- les identifiants locaux du dashboard Netdata ne sont pas présents sur ce poste ;
- le webhook de notification GitHub Actions n'est pas configuré dans les secrets Actions listés ;
- le monitoring Netdata/Discord est documenté comme actif directement sur le VPS.

Le champ `environment` du healthcheck staging vaut actuellement `production`, car le conteneur est
exécuté avec un runtime optimisé production. Ce champ ne permet donc **pas** de distinguer staging de
la vraie production. Le garde-fou doit vérifier le hostname, le SHA attendu, les identifiants des
fixtures et, si nécessaire, l'empreinte de la base staging.

Les workflows et scripts actuellement présents dans `attendee-ems-back` sont la source de vérité
opérationnelle. Les anciens documents citant `staging-up.sh`, un endpoint `/health` sans préfixe ou
des automatismes encore non prouvés doivent être réaudités avant utilisation.

## 2. Contrainte structurelle

La stack staging est isolée par Docker mais partage le même VPS physique que la production. Ses
conteneurs ont des limites CPU/RAM, mais une forte charge peut encore consommer réseau, disque et I/O
partagés.

Conséquences :

- aucun test élevé sans surveillance simultanée de staging et des ressources partagées ;
- aucune modification d'un conteneur, volume, réseau ou service de production ;
- aucun reboot du VPS pendant la session autonome ;
- si la production se dégrade, arrêter immédiatement k6 et ne pas intervenir dessus ;
- si la surveillance en lecture seule de la santé générale du VPS/prod n'est pas autorisée, ne pas
  lancer le scénario maximal sur cette infrastructure partagée.

## 3. Watchdog obligatoire avant tout test élevé

Avant de construire le watchdog, les anciens scripts k6 peuvent servir de référence de lecture, mais
ils ne doivent pas être exécutés directement ni copiés en bloc depuis l'archive. Le runner final doit
être versionné, relu, testé avec une charge minimale et capable de produire un rapport après arrêt.

Le lanceur de test doit :

1. refuser toute cible autre que le hostname staging explicitement autorisé ;
2. vérifier le SHA staging attendu et l'état DB/Redis/migrations ;
3. enregistrer le PID exact du processus k6 ;
4. sonder le healthcheck staging pendant le test ;
5. surveiller CPU, RAM, réseau et état Docker du générateur et du staging ;
6. arrêter uniquement ce PID k6 par signal contrôlé afin de conserver son résumé ;
7. conserver logs, timestamps, métriques et état DB avant tout nettoyage ;
8. ne jamais relancer automatiquement une charge après incident.

Déclencheurs d'arrêt automatique initiaux :

- trois échecs consécutifs du healthcheck staging ;
- hausse soutenue des `5xx` inattendus au-dessus du seuil de sécurité du scénario ;
- violation d'intégrité : surbooking, compteur incohérent ou doublon non maîtrisé ;
- générateur k6 saturé ou incapable d'injecter la charge annoncée ;
- conteneur staging redémarré/OOM/unhealthy ;
- signe de dégradation des ressources partagées ou de la production observé en lecture seule.

Les valeurs précises d'erreur/latence sont définies par scénario. Une violation d'intégrité interrompt
toujours immédiatement la recherche de capacité.

## 4. Séquence de récupération

### Niveau 0 — ralentissement sans panne

1. arrêter k6 ;
2. laisser le staging drainer pendant une période de refroidissement ;
3. capturer p95/p99, files, connexions DB et ressources ;
4. contrôler les invariants métier ;
5. diagnostiquer avant tout nouveau palier.

### Niveau 1 — processus API staging tombé

Le `restart: unless-stopped` doit normalement faire redémarrer le conteneur après la sortie du
processus.

1. arrêter k6 ;
2. attendre et sonder le healthcheck ;
3. vérifier en lecture seule l'état et les logs de `ems-staging-api` ;
4. si Docker ne l'a pas rétabli et si l'autorisation de recovery est confirmée, redémarrer **une fois**
   `ems-staging-api` uniquement ;
5. vérifier health, version et migrations ;
6. ne pas reprendre la charge sans diagnostic et nouveau rapport.

### Niveau 2 — API vivante mais unhealthy

Docker ne redémarre pas nécessairement un processus encore vivant uniquement parce que son
healthcheck est rouge.

1. arrêter k6 ;
2. capturer état et logs ;
3. tenter au maximum un restart de l'API staging si DB et Redis staging sont sains ;
4. sinon, ne pas redémarrer aveuglément PostgreSQL/Redis ;
5. utiliser le CD staging sur le même SHA uniquement si un redéploiement est justifié et autorisé ;
6. arrêter la campagne si la santé complète ne revient pas.

### Niveau 3 — dépendance staging ou données en danger

- arrêter la charge ;
- ne lancer aucun nettoyage ;
- ne supprimer aucun volume ;
- ne restaurer aucune sauvegarde automatiquement ;
- conserver les preuves et marquer la campagne bloquée jusqu'à décision humaine.

### Niveau 4 — VPS/SSH/Docker injoignable

Le CD GitHub Actions dépend lui aussi de SSH et ne peut pas réparer un hôte totalement injoignable.
Comme le VPS héberge aussi la production :

- ne déclencher aucun reboot global de façon autonome ;
- interrompre la session de charge ;
- consigner le dernier état connu et l'incident ;
- laisser le monitoring externe alerter le canal existant ;
- attendre l'intervention de la personne autorisée côté OVH.

Ce niveau reste un vrai blocage tant que la haute disponibilité est reportée.

## 5. Actions strictement interdites

- commande Docker ciblant `ems-api`, `ems-postgres`, `ems-redis`, `ems-nginx` ou tout autre composant
  de production ;
- `docker compose down`, et surtout `down -v`, pendant un incident ;
- reboot du VPS ;
- restauration ou suppression de volume ;
- déploiement `main`/production ;
- reprise automatique au même palier après une panne ;
- masquer un redémarrage ou une erreur dans le rapport de résultat.

## 6. Autorisations à inclure dans le GO de session

Pour une exécution réellement autonome en l'absence du responsable, le GO doit autoriser :

- l'installation locale de k6 et des outils de mesure nécessaires ;
- la surveillance **en lecture seule** de la santé et des ressources partagées/prod pendant la charge ;
- l'arrêt automatique du test dès qu'un garde-fou se déclenche ;
- un redémarrage unique de `ems-staging-api` si nécessaire ;
- le redéploiement du SHA staging courant via le CD staging si le simple restart ne suffit pas ;
- la poursuite des travaux locaux/CI non dangereux après l'arrêt de la campagne de charge.

Ces autorisations n'incluent jamais une mutation de la production, un reboot du VPS, un restart de
PostgreSQL/Redis, une restauration ou une suppression de volume.

## 7. Continuité de la session

Une panne staging ne doit pas perdre le travail : avant chaque déploiement ou campagne, tout élément
non sensible terminé est commité et poussé. Après incident, la session peut continuer sur les reviews,
tests locaux, corrections et documentation qui ne nécessitent pas le staging.

La session ne peut toutefois pas être garantie « sans interruption » si le VPS, GitHub, le réseau,
les credentials ou l'environnement d'exécution deviennent indisponibles. Dans ce cas, le dernier
checkpoint Git constitue le point de reprise vérifiable.
