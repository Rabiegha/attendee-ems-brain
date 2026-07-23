# Suivi opérationnel — Gotenberg Cloud Run + Auth OIDC

> Chantier B0/B1 — LFD 2026
> Dernière vérification réelle : **23/07/2026 à 10h47 Paris**
> Objet du call : confirmer l'exploitation GCP, puis organiser les derniers tests métier et de charge.

## Résumé à donner pendant le call

Le chemin technique **OVH staging → OIDC → Cloud Run privé → Gotenberg → PDF** fonctionne.

- B0 et B1 sont mergés dans `main` et `staging`, avec CI verte.
- Le staging OVH exécute l'image immutable `ac5b01a072251a4e6c64934747ce92faae7eae73`.
- `GOTENBERG_ENABLED=true` sur staging ; Gotenberg n'est pas activé en production.
- Le credential est hors Git, mode `600`, monté en lecture seule dans le conteneur API.
- Le service Cloud Run refuse toujours les appels anonymes en `403`.
- Le smoke lancé le 23/07 depuis le conteneur staging réellement déployé est vert :
  - health authentifié : OK ;
  - conversion HTML : OK ;
  - durée : **271 ms** ;
  - PDF : **8 117 octets**, signature `%PDF`.
- Le client Gotenberg est branché sur les trois chemins PDF et Puppeteer reste le fallback
  automatique.
- Le flux métier « inscription session → vrai badge PDF → pièce jointe email » est codé, mergé et
  déployé en staging, mais son smoke bout en bout avec un vrai événement LFD reste à exécuter.

## État réel vérifié

| Sujet | État au 23/07 | Preuve |
| --- | --- | --- |
| Cloud Run Gotenberg | ✅ Accessible | URL réelle répond ; endpoint privé |
| Auth anonyme | ✅ Refusée | `GET /health` et endpoint conversion → `403` sans token |
| Auth OIDC depuis OVH | ✅ Validée | health OIDC OK depuis `ems-staging-api` |
| Binding IAM | ✅ Validé fonctionnellement | token du credential accepté par Cloud Run |
| Conversion PDF minimale | ✅ Validée | 8 117 octets, `%PDF`, 271 ms |
| Client Nest B0 | ✅ Mergé/déployé | PR #51 + correctif multipart PR #54 |
| Branchement B1 | ✅ Mergé/déployé | trois points PDF, retries, fallback, logs/métriques |
| Credential staging API | ✅ Présent | fichier mode `600`, montage `/run/secrets:ro` |
| Gotenberg staging | ✅ Activé | `GOTENBERG_ENABLED=true` |
| Gotenberg production | ⏸️ Désactivé/non configuré | Puppeteer reste le chemin actif |
| Vrai badge LFD sur staging | 🟠 À valider | smoke minimal vert, pas encore de preuve métier récente |
| Email avec PDF joint | 🟠 Codé, smoke réel restant | service et tests présents, aucun run métier observé |
| Fallback réel provoqué | 🟠 À valider | couvert par tests, pas encore provoqué sur staging |
| Charge 3 000 PDF | 🔴 Non exécutée | runner sur branche dédiée, aucun rapport de résultat |
| Credential dans le worker | 🔴 Absent | volume secrets monté sur l'API seulement |

## Configuration confirmée

| Paramètre | Valeur |
| --- | --- |
| Projet GCP | `attendee-494112` |
| Région | `europe-west9` (Paris) |
| URL Cloud Run | `https://badge-generator-service-ipn4gzbe6a-od.a.run.app` |
| Audience OIDC | même URL, sans chemin ni slash final |
| Compte de service | `gotenberg-invoker@attendee-494112.iam.gserviceaccount.com` |
| Endpoint santé | `GET /health` |
| Endpoint PDF | `POST /forms/chromium/convert/html` |
| Timeout applicatif | `60 s` |
| Limite annoncée | `32 MB` par requête |

L'ancienne adresse sans tiret dans le project ID était erronée. Le smoke authentifié prouve que
l'identité avec `attendee-494112` est bien autorisée.

## Smoke réel du 23 juillet

Exécuté dans `ems-staging-api`, et non depuis un poste local :

```txt
[1/2] GET .../health (token OIDC)
Health check OK.

[2/2] POST .../forms/chromium/convert/html
Gotenberg PDF généré: durationMs=271 pdfBytes=8117
signature=%PDF
Smoke test B0: SUCCÈS
```

Contrôle anonyme effectué séparément le même jour :

```txt
GET /health                              -> 403
POST /forms/chromium/convert/html        -> 403 sans token
```

Ce test valide le réseau sortant OVH, le credential, l'audience, IAM, le multipart et le retour PDF.
Il ne valide pas encore un template LFD complet, les polices, R2 ni l'email final.

## Ce qui est maintenant dans le code

### B0 — client OIDC privé

- `google-auth-library.getIdTokenClient(audience)` ;
- cache et renouvellement du token gérés par la bibliothèque ;
- `GET /health` authentifié ;
- envoi multipart avec fichier principal nommé `index.html` ;
- validation d'une réponse non vide commençant par `%PDF` ;
- erreurs structurées : timeout, indisponibilité, 401/403, configuration désactivée ;
- aucun token, credential ou HTML métier dans les logs.

Un défaut réel de boundary multipart a été découvert pendant le premier smoke. Le correctif fige
le formulaire en `Buffer`, conserve la boundary et fixe `content-length`. Le smoke du 23/07 valide
ce correctif depuis l'image déployée.

### B1 — moteur principal + fallback

Les trois chemins PDF utilisent Gotenberg lorsque le flag est actif :

1. export PDF de test depuis l'éditeur ;
2. PDF multi-pages d'un événement ;
3. PDF d'un badge individuel.

Politique actuelle :

- 1 essai initial + 2 retries sur timeout/indisponibilité ;
- backoff `500 ms`, puis `1 500 ms` ;
- aucun retry sur 401/403 ou mauvaise configuration ;
- fallback Puppeteer automatique après échec ;
- Puppeteer direct si `GOTENBERG_ENABLED=false` ;
- logs structurés `pdf_render` et compteurs en mémoire.

Il existe **27 tests ciblés** sur le client, les erreurs, les métriques, le retry/fallback et le
câblage des trois points de rendu.

### Email avec badge PDF

Le service `SessionRegistrationEmailService` :

1. génère ou récupère le badge ;
2. demande le PDF au moteur Gotenberg/Puppeteer ;
3. ajoute `badge-<registrationId>.pdf` comme pièce jointe ;
4. passe le message à l'infrastructure email avec clé d'idempotence.

Ce code est présent dans le SHA staging actuel. Il manque encore un smoke métier complet avec un
vrai événement, un vrai template et une boîte de réception de test.

## PR et déploiements

| PR | Contenu | État |
| --- | --- | --- |
| #51 | Client OIDC Gotenberg B0 | ✅ mergée, CI verte |
| #53 | B1 : branchement, retries et fallback Puppeteer | ✅ mergée, CI verte |
| #54 | Correctif boundary multipart | ✅ mergée, CI verte |
| #56 | Montage du secret Gotenberg sur l'API staging | ✅ mergée, CI verte |
| #57 | Scripts de diagnostic Gotenberg/Puppeteer | ✅ mergée, CI verte |

Le SHA staging `ac5b01a` a lui aussi une CI complète verte et correspond au SHA déployé.

## Environnements

### Staging

```txt
GOTENBERG_ENABLED=true
GOTENBERG_BASE_URL=https://badge-generator-service-ipn4gzbe6a-od.a.run.app
GOOGLE_APPLICATION_CREDENTIALS=/run/secrets/gotenberg-invoker-key.json
```

- API healthy ;
- credential présent dans l'API ;
- secret monté en lecture seule ;
- Gotenberg réellement joignable et authentifié.

### Production

- code B0/B1 présent ;
- variables Gotenberg absentes/non activées ;
- comportement actuel : Puppeteer ;
- aucune activation prod avant le smoke métier staging, le test de fallback et une décision GO.

## Point découvert : worker sans credential

Le conteneur `ems-staging-worker` reçoit les variables Gotenberg, mais le volume
`/opt/ems-attendee/staging/secrets:/run/secrets:ro` n'est monté que dans `ems-staging-api`.

Conséquence :

- pas de blocage immédiat pour le rendu synchrone encore exécuté dans l'API ;
- blocage futur dès que B2 déplace la génération PDF dans le worker ;
- en l'état, le worker tenterait Gotenberg sans fichier de credential puis basculerait vers
  Puppeteer, ce qui irait à l'encontre de l'architecture cible et pourrait charger le VPS.

Action avant B2 : monter le même secret read-only dans le worker, redéployer, puis refaire le smoke
depuis le worker.

## Ce qui reste réellement à faire

### P0 avant activation production

- [ ] Générer un badge avec le template LFD réel depuis staging.
- [ ] Vérifier visuellement dimensions, QR, accents, images et polices.
- [ ] Tester le flux complet inscription session → PDF → email avec pièce jointe.
- [ ] Provoquer une indisponibilité contrôlée et prouver le fallback Puppeteer.
- [ ] Vérifier que les logs `pdf_render` permettent d'identifier moteur, durée, taille et fallback.
- [ ] Décider explicitement du GO production.

### P1 capacité et exploitation

- [ ] Faire valider par l'équipe GCP la valeur actuelle de `min-instances`.
- [ ] Confirmer l'image Gotenberg exacte et son digest, pas seulement `gotenberg:8`.
- [ ] Confirmer les limites Cloud Run : concurrence, max instances, CPU, mémoire et timeout.
- [ ] Exécuter une montée progressive avant les 3 000 PDF :
  `10 → 100 → 500 → 1 000 → 3 000`.
- [ ] Définir les seuils d'arrêt : erreurs, 429/5xx, p95, saturation et coût.
- [ ] Vérifier Cloud Run Logs/Monitoring et prévoir les alertes.

### P1 worker/B2

- [ ] Monter le credential read-only dans `ems-staging-worker`.
- [ ] Refaire health + PDF depuis ce conteneur.
- [ ] Séparer la génération PDF de l'API avec concurrence Puppeteer bornée.

### P2 secrets

- [ ] Conserver la clé hors Git et restreinte à `roles/run.invoker` sur ce service.
- [ ] Définir rotation/révocation.
- [ ] Étudier Secret Manager ou Workload Identity Federation après l'événement pour éviter une clé
  longue durée sur OVH.

## Questions à poser à l'équipe GCP

1. Quelle image/digest Gotenberg est actuellement déployée ?
2. Quelle est la valeur actuelle de `min-instances` ? Peut-on confirmer `1` pendant les pics LFD ?
3. Quelles sont les valeurs `concurrency`, `max-instances`, CPU et mémoire ?
4. Pouvez-vous retrouver dans Cloud Run Logs le smoke du **23/07 à 08:47:11 UTC** ?
5. Voyez-vous une latence ou un cold start anormal sur ce smoke de 271 ms ?
6. Quels quotas ou protections risquent de limiter une journée à 3 000 PDF ?
7. Quelles métriques et alertes devons-nous activer avant le test de charge ?
8. Confirmez-vous que le rôle du service account reste limité à ce seul service Cloud Run ?
9. Souhaitez-vous imposer une rotation de la clé avant l'événement ?
10. Y a-t-il une contrainte d'egress pour les Google Fonts/images externes utilisées par Chromium ?

Formulation courte en anglais :

```txt
The private OIDC flow from our real OVH staging container is now working.
Today at 08:47:11 UTC, health succeeded and Gotenberg returned an 8,117-byte PDF in 271 ms.

Could you please confirm the deployed image digest, min/max instances, concurrency,
CPU/memory, and whether you can see this exact request in Cloud Run Logs?
We also need your recommended monitoring and stop thresholds before ramping up to 3,000 PDFs.
```

## Déroulé conseillé du call

1. **2 min — état** : montrer le smoke vert OVH → OIDC → Cloud Run.
2. **5 min — exploitation** : image, scaling, concurrence, quotas, logs et alertes.
3. **3 min — charge** : faire valider la rampe et les seuils d'arrêt.
4. **3 min — sécurité** : scope IAM, rotation et cible Secret Manager/WIF.
5. **2 min — décisions** : responsables et dates pour smoke métier, charge et GO production.

## Critère de sortie du prochain jalon

Le chantier est prêt pour une activation production lorsque :

- un badge LFD réel est conforme ;
- l'email réel contient le bon PDF ;
- le fallback Puppeteer est prouvé ;
- la capacité et les seuils Cloud Run sont validés ;
- le secret est disponible dans le futur worker ;
- une décision humaine explicite autorise l'activation.
