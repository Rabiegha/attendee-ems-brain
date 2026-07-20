# B2 — Fallback PDF Puppeteer résilient et asynchrone

> **Statut :** cadré le 20/07/2026 — à démarrer après B1, en réutilisant la file PDF de C2.1.
> **Priorité :** 🟠 P1 résilience LFD. **Estimation brute :** ~2–4 j ; **incrément net :**
> ~1–2 j si C2.1 livre déjà `pdf.generate` et le worker séparé.
> **Dépendances :** B1 (moteurs Gotenberg/Puppeteer), C2.1 (orchestration après inscription),
> 0-MON (alertes) et T4 (tests PDF/badges).

## Décision

Gotenberg reste le moteur principal. Puppeteer local reste un filet de sécurité, mais il ne doit
plus être lancé sans contrôle dans le processus qui répond aux inscriptions ou aux téléchargements.

Le fallback cible est donc :

```txt
inscription confirmée et transaction DB commitée
-> job durable pdf.generate (clé = registration_id + type d'artefact)
-> worker PDF séparé
-> Gotenberg (primaire, retries bornés)
-> si panne éligible et circuit ouvert : lane Puppeteer limitée
-> validation du PDF puis stockage R2
-> badge.pdf_url + status=completed
-> email.send
```

L'inscription reste réussie même si les deux moteurs PDF sont indisponibles. Le billet reste en
`pending/failed_retryable`, visible comme « en préparation », et sera repris par la file.

## Audit de l'existant au 20/07/2026

B1 est implémenté sur la branche backend `feat/pdf-generation-hardening` (commit `9ed7321`),
au-dessus de B0 déjà présent sur `staging` (`f37df97`). Il couvre déjà une partie importante :

| Déjà couvert par B1 | Limite restante que B2 traite |
| --- | --- |
| Gotenberg primaire et Puppeteer automatique en fallback | Le fallback est exécuté inline dans le même process/requête |
| 3 tentatives Gotenberg, backoff 500 ms puis 1 500 ms | Timeout Gotenberg par défaut 60 s : l'attente peut approcher 3 min avant Puppeteer |
| Retry uniquement sur erreurs transitoires | Pas de circuit breaker : une panne Gotenberg déclenche la même séquence pour chaque requête |
| Logs structurés et compteurs in-process | Compteurs perdus au restart et non raccordés à une alerte |
| Browser Puppeteer réutilisé | Pas de limite globale de pages/rendus concurrents ni de lane isolée |
| Cache R2 réutilisé quand `pdf_url` est disponible | Un miss R2 régénère encore à la volée dans l'endpoint |

Les trois points de rendu actuels (`generateTestBadge`, `generateEventBadgesPDF`,
`getBadgePdfBase64`) attendent tous `renderPdfWithFallback()`. Il n'existe pas encore de queue PDF
dans `QUEUE_NAMES`; BullMQ est déjà utilisé pour les exports et `email.send`.

Autre garde-fou concret : les pages Puppeteer sont actuellement fermées seulement sur le chemin
heureux. B2 doit utiliser un `finally` afin qu'une erreur de chargement/police/PDF ne laisse pas une
page Chromium orpheline.

Sources code auditées :

- [badge-generation.service.ts](../../../../../attendee-ems-back/src/modules/badge-generation/badge-generation.service.ts)
- [gotenberg-client.service.ts](../../../../../attendee-ems-back/src/modules/gotenberg/gotenberg-client.service.ts)
- [queue-names.ts](../../../../../attendee-ems-back/src/infra/queue/queue-names.ts)
- [runbook B1](../../../../../attendee-ems-back/docs/runbooks/pdf-generation-hardening-b1.md)

## Pourquoi le fallback inline est dangereux

Pendant une panne Gotenberg, plusieurs inscriptions simultanées peuvent toutes :

1. garder une requête Node ouverte pendant les retries ;
2. atteindre presque ensemble le fallback ;
3. ouvrir plusieurs pages Chromium sur le VPS ;
4. consommer CPU/RAM et ralentir l'inscription, Redis, Postgres et le worker email.

Le filet de sécurité devient alors un amplificateur de panne. Le but de B2 n'est pas d'ajouter
encore des retries à B1, mais d'isoler et de borner ce travail.

## Contrat métier retenu

- Un QR identifie la `registration` de l'attendee à l'événement.
- Le contrôle d'accès à une session est dynamique au scan.
- Il n'y a donc pas besoin d'un `ticket_id` métier séparé pour déclencher le PDF.
- Il n'y a pas de `ticket_version` métier pour le MVP tant que le PDF ne liste pas les sessions.
- Le record `Badge`, déjà unique par `registration_id`, peut porter `status`, `pdf_url` et
  `error_message`.
- La clé d'idempotence recommandée est `pdf:badge:<registrationId>`. Si le contenu imprimé devient
  mutable plus tard, ajouter une empreinte technique de contenu/template, pas une version de QR.

## Architecture cible minimale

### 1. Une file durable et un worker séparé

- Ajouter `pdf.generate` à l'infrastructure BullMQ existante.
- Enqueue seulement après le commit de l'inscription ; l'outbox transactionnelle complète reste
  un approfondissement ultérieur si l'enqueue après commit ne suffit pas au risque accepté LFD.
- Déployer le processor PDF dans un rôle/process distinct de l'API avant le test de volume.
- Utiliser un `jobId` déterministe dérivé de `registration_id` et du type d'artefact.
- Ne jamais joindre le PDF à `email.send` avant que R2 et le statut `completed` soient confirmés.

### 2. Deux lanes avec budgets différents

| Lane | Politique |
| --- | --- |
| Gotenberg primaire | Concurrence configurable, timeout/retries bornés, priorité normale |
| Puppeteer fallback | Concurrence très basse (1 au départ), timeout strict, budget/minute et file bornée |

Les billets individuels issus d'une inscription sont prioritaires. Les previews admin et exports
multi-badges ne doivent pas saturer la lane de secours ; ils sont ralentis, mis dans une priorité
basse, ou refusés temporairement avec un message explicite.

### 3. Circuit breaker Gotenberg

- Fermer le circuit en fonctionnement normal.
- L'ouvrir après un seuil configurable d'échecs transitoires dans une fenêtre glissante.
- Circuit ouvert : ne plus attendre trois timeouts pour chaque job ; envoyer les jobs éligibles vers
  la lane Puppeteer dans la limite de son budget.
- Après un cooldown, passer en `half-open` et tester un canari Gotenberg.
- Refermer seulement après succès du canari ; sinon prolonger le cooldown.
- Une erreur de template/données/PDF invalide n'est pas une panne de moteur et ne doit pas ouvrir le
  circuit global.

L'état partagé du circuit et les compteurs de budget vivent dans Redis, avec comportement
fail-closed : si Redis est indisponible, ne pas lancer une rafale Puppeteer ; garder les jobs en file.

### 4. Retry et classification

| Erreur | Action |
| --- | --- |
| Timeout/réseau/5xx Gotenberg | Retry borné, puis fallback si budget disponible |
| 401/403/config Gotenberg | Pas de retry en boucle ; alerte immédiate, fallback contrôlé |
| HTML/données/template invalides | Échec permanent, pas de fallback aveugle |
| Crash/disconnexion Chromium | Recréer le browser une fois, puis retry BullMQ |
| Timeout page/police/image | Fermer la page en `finally`, classer retryable selon le cas |
| R2 indisponible | Ne pas publier `completed`; retry de stockage sans refaire le rendu si possible |

Le nombre d'essais B1 et le nombre d'essais BullMQ doivent former un budget total explicite. Il ne
faut pas multiplier implicitement « 3 essais moteur × 3 essais job » sans limite temporelle.

### 5. Publication atomique de l'artefact

1. produire dans un buffer/fichier temporaire ;
2. vérifier taille non nulle et signature `%PDF` ;
3. uploader vers une clé R2 déterministe ;
4. mettre à jour `pdf_url`, `generated_at`, `status=completed` dans une transaction courte ;
5. seulement ensuite enfiler l'email idempotent.

Un job obsolète ne doit jamais écraser un artefact plus récent. Pour le MVP sans version métier,
l'unicité par registration et le lock de job suffisent ; si le template devient modifiable pendant
la génération, comparer une empreinte de contenu avant publication.

## UX et API

- Après inscription : réponse immédiate avec billet `pending` et URL de suivi C2.1.
- Si le PDF R2 existe : téléchargement immédiat.
- S'il manque : enqueue idempotent et réponse `202`/statut `generating`, jamais rendu Puppeteer
  public synchrone sous charge.
- Après épuisement des retries : message « billet temporairement indisponible » avec possibilité de
  reprise opérateur ; l'inscription et le QR restent valides.

## Lots d'implémentation

### B2.0 — Fermer les fuites et borner Puppeteer (~0,5 j)

- `page.close()` en `finally` sur les trois chemins ;
- verrou de lancement du browser et récupération après crash ;
- sémaphore global Puppeteer, concurrence par défaut `1` ;
- timeout de rendu et métriques durée/échec/saturation.

### B2.1 — Asynchroniser le billet critique (~1–2 j, partagé avec C2.1)

- queue `pdf.generate`, job idempotent et états `pending/generating/completed/failed` ;
- worker séparé de l'API ;
- publication R2 puis déclenchement `email.send` ;
- endpoint de statut/polling et comportement `202` sur cache miss.

### B2.2 — Circuit breaker et exploitation (~0,5–1 j)

- circuit partagé Redis, cooldown et canari ;
- priorités/budgets de fallback ;
- alertes file, taux de fallback, durée et saturation ;
- commandes pause/reprise/force-Gotenberg/disable-Puppeteer documentées.

## Definition of Done B2

- [ ] Une inscription ne lance et n'attend jamais Chromium/Gotenberg dans sa requête HTTP.
- [ ] Le worker PDF est isolé du rôle API et redémarrable sans perdre les jobs.
- [ ] Un même `registration_id` ne produit pas plusieurs jobs ou emails concurrents.
- [ ] Puppeteer a une concurrence et un budget explicites ; saturation = attente, pas surcharge API.
- [ ] Les pages Chromium sont fermées en `finally` et un crash browser est récupéré proprement.
- [ ] Le circuit breaker évite les trois timeouts répétés pendant une panne Gotenberg.
- [ ] Le statut n'est `completed` qu'après validation du PDF et stockage R2 réussi.
- [ ] Tests : succès Gotenberg, panne longue, fallback saturé, crash Chromium, R2 KO, retry/restart
  worker, job dupliqué et priorité billet vs export.
- [ ] Test de charge avec Gotenberg forcé KO : latence inscription stable et CPU/RAM VPS sous seuil.
- [ ] Alerte opérationnelle sur profondeur/âge de file, taux de fallback et échecs définitifs.

## Hors périmètre B2

- inventer une nouvelle identité de billet ou un `ticket_id` ;
- versionner le QR à chaque changement de session ;
- remplacer Gotenberg par Puppeteer comme moteur nominal ;
- architecture event-driven globale/outbox générique (chantier L) ;
- améliorer le fallback email SMTP (chantier C4).
