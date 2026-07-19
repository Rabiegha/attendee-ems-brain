# Entrées de travail B0/B1 — Gotenberg privé

> **Date de collecte :** 2026-07-20
>
> **Classification :** configuration non secrète, versionnable
>
> **Secret associé :** clé JSON du service account, strictement hors Git

## 1. But

Conserver les paramètres nécessaires à l'intégration Nest → Gotenberg Cloud Run privé afin que le
chantier B0/B1 puisse reprendre depuis un autre poste sans dépendre de l'historique de conversation.

Ce fichier ne contient ni clé privée, ni jeton OIDC, ni contenu du credential JSON.

## 2. Configuration confirmée

| Paramètre | Valeur |
| --- | --- |
| Projet GCP | `attendee-494112` |
| Région | `europe-west9` (Paris) |
| URL Cloud Run | `https://badge-generator-service-ipn4gzbe6a-od.a.run.app` |
| Audience OIDC | `https://badge-generator-service-ipn4gzbe6a-od.a.run.app` |
| Compte de service appelant | `gotenberg-invoker@attendee-494112.iam.gserviceaccount.com` |
| Endpoint PDF | `POST /forms/chromium/convert/html` |
| Endpoint santé | `GET /health` |
| Timeout demandé | `60 s` |
| Taille maximale d'une requête | `32 MB` |

L'audience est exactement l'URL Cloud Run, sans slash final ni chemin d'endpoint.

Le service est privé (`--no-allow-unauthenticated`). Une requête anonyme doit être rejetée par le
frontend Google avant d'atteindre Gotenberg. Le rôle `roles/run.invoker` est annoncé comme limité à
ce service.

## 3. Contrat de l'appel Nest

- utiliser `google-auth-library` et
  `getIdTokenClient("https://badge-generator-service-ipn4gzbe6a-od.a.run.app")` ;
- laisser la bibliothèque mettre en cache et renouveler le token OIDC ;
- ne développer aucune logique maison de stockage/rafraîchissement du token ;
- envoyer le HTML en multipart avec la partie principale nommée `index.html` ;
- joindre CSS, images et polices en fichiers supplémentaires référencés par chemin relatif ;
- appliquer un timeout explicite de 60 secondes ;
- mesurer la taille totale du HTML et des ressources avant le plafond de 32 MB ;
- ne journaliser ni token, ni clé, ni payload contenant des données personnelles.

## 4. Secret requis — jamais versionné

Le credential est un fichier JSON de type `service_account`. Sa présence et sa structure ont été
vérifiées localement le 2026-07-20, sans lecture ni affichage de la clé privée.

Règles impératives :

- ne jamais copier le JSON dans ce dépôt, un commit, une PR, un ticket ou une conversation ;
- restreindre ses permissions locales au propriétaire (`chmod 600`) ;
- sur staging, le poser par le mécanisme sécurisé de l'environnement OVH ;
- préférer un fichier monté et référencé par `GOOGLE_APPLICATION_CREDENTIALS` à une variable JSON
  multiligne ;
- confirmer que `.gitignore` couvre toute destination locale éventuelle avant la première exécution ;
- transférer le credential vers un autre poste uniquement par le canal sécurisé prévu ;
- révoquer/remplacer immédiatement la clé en cas d'exposition suspectée.

La migration vers Secret Manager reste un travail post-event. Cela ne rend jamais acceptable le
commit temporaire du credential.

## 5. Point de cohérence IAM à vérifier

Le message de transmission mentionnait une fois
`gotenberg-invoker@attendee494112.iam.gserviceaccount.com` sans tiret. Le credential reçu contient :

`gotenberg-invoker@attendee-494112.iam.gserviceaccount.com`

Cette seconde valeur correspond au `project_id` du credential. Avant de conclure B0, vérifier par un
smoke test que `roles/run.invoker` est bien accordé à cette identité exacte. Un `403` avec un token
correct doit conduire à contrôler ce binding en premier.

## 6. Smoke tests obligatoires depuis Nest

1. `GET /health` sans token → `403` attendu.
2. `GET /health` avec token OIDC et audience exacte → succès attendu.
3. génération d'un PDF d'une page → réponse non vide commençant par `%PDF`.
4. requête visible dans les logs Cloud Run, sans secret ni donnée personnelle inutile.
5. audience incorrecte → refus attendu.
6. timeout/indisponibilité simulée → erreur Nest bornée et exploitable.
7. test C2.1 → réservation rapide, génération PDF asynchrone, puis envoi email.

Les quatre premiers contrôles ont déjà été annoncés comme validés côté service. B0 doit reproduire
la validation de bout en bout depuis le backend Nest réellement déployé sur staging.

## 7. Décision warm/cold

Décision recommandée :

- conserver `min-instances=1` pendant l'intégration, les tests de charge et les fenêtres de pointe ;
- conserver `min-instances=1` pendant les journées de l'événement ;
- passer à `min-instances=0` pendant les périodes calmes si le coût idle doit être évité ;
- ne modifier cette configuration que par la personne autorisée côté GCP.

La configuration annoncée au 2026-07-20 est `min-instances=1`.

## 8. Reprise depuis un autre poste

Avant de reprendre B0/B1 ailleurs :

- cloner/fetcher la branche et vérifier le dernier SHA publié ;
- obtenir le credential par canal sécurisé ou confirmer qu'il est déjà injecté dans staging ;
- ne pas recopier l'ancien chemin absolu local dans le code ;
- configurer le chemin du credential uniquement dans l'environnement local/staging ;
- vérifier `git status` avant et après la configuration du secret ;
- relire cette fiche et la spec principale de session avant toute modification.

La documentation, le code, les tests et les rapports non sensibles doivent être commités et poussés.
Le credential, les tokens et leurs valeurs restent toujours hors Git et doivent suivre un canal de
transfert séparé.
