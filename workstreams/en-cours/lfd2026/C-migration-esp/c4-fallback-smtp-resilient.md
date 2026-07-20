# C4 — Fallback d'envoi SMTP résilient et sous quota

> **Statut :** cadré le 20/07/2026 — à implémenter après stabilisation de C2/C3.
> **Priorité :** 🟠 P1 résilience LFD. **Estimation :** ~2–3 j, hors délai éventuel pour
> confirmer/adapter l'offre SMTP OVH.
> **Dépendances :** C2 (`email.send`, idempotence, webhooks), C3 (délivrabilité et règles de
> warm-up), 0-MON (alertes) et validation contractuelle du quota SMTP.

## Décision

Mailgun API EU reste le transport primaire. Le SMTP OVH historique reste disponible comme filet de
sécurité, mais jamais comme bascule aveugle « si Mailgun jette une erreur, envoyer tout de suite via
SMTP ».

Le fallback doit protéger les emails transactionnels critiques sans :

- doubler un email dont l'acceptation Mailgun est incertaine ;
- vider le quota SMTP en quelques minutes ;
- envoyer les campagnes/warm-up sur un domaine non préparé pour ce volume ;
- transformer une panne provider en surcharge Redis/API/SMTP ;
- annoncer « délivré » alors que le serveur SMTP a seulement accepté le message.

## Audit de l'existant au 20/07/2026

### Ce qui existe déjà

- `EMAIL_QUEUE_ENABLED=true` en staging et production : l'appelant enqueue et ne bloque pas sur
  Mailgun/SMTP.
- BullMQ `email.send` avec 3 tentatives et backoff exponentiel de base 5 s.
- Processor à concurrence 5.
- Throttle Redis pilotable à chaud (`rate_limit_per_second`, pause globale).
- `jobId` déterministe quand une clé d'idempotence métier est fournie.
- Transport Mailgun API EU et transport SMTP Nodemailer derrière la même interface.
- Événements Mailgun signés/persistés pour `delivered`, `failed`, bounce, etc.

### Ce qui n'existe pas encore

- SMTP n'est **pas** un fallback : `EMAIL_TRANSPORT` choisit un seul provider au démarrage et une
  bascule demande une modification de configuration puis un restart.
- Le processor ne classifie pas finement 429, 5xx, timeout ambigu, 401/403, rejet destinataire ou
  erreur permanente ; toutes les erreurs remontées suivent les mêmes retries BullMQ.
- Pas de circuit breaker, de cooldown, de canari de retour Mailgun ni de mode manuel/automatique.
- Pas de quota/jour ni de budget réservé au SMTP fallback.
- Pas de ledger de tentatives par provider permettant un handoff sûr.
- SMTP n'a pas les webhooks Mailgun : son `messageId` et `accepted` ne prouvent pas la livraison.
- Nodemailer n'a pas encore de timeouts explicites, de pool borné ni de métriques provider.

Configuration réelle vérifiée sans lire de secret :

```txt
staging    EMAIL_TRANSPORT=mailgun · EMAIL_QUEUE_ENABLED=true · SMTP_HOST=mailpit
production EMAIL_TRANSPORT=mailgun · EMAIL_QUEUE_ENABLED=true · SMTP_HOST=ssl0.ovh.net
```

`ssl0.ovh.net` confirme un relais mail OVH compatible MX Plan, mais la limite quotidienne exacte
ne peut pas être déduite du hostname ou du code. Elle dépend de l'offre/du compte : **aucun chiffre
ne doit être hardcodé avant vérification dans l'espace client ou auprès d'OVH**.

Sources code auditées :

- [email.module.ts](../../../../../attendee-ems-back/src/modules/email/email.module.ts)
- [email-send.processor.ts](../../../../../attendee-ems-back/src/modules/email/queue/email-send.processor.ts)
- [email-throttle.service.ts](../../../../../attendee-ems-back/src/modules/email/queue/email-throttle.service.ts)
- [smtp.transport.ts](../../../../../attendee-ems-back/src/modules/email/transport/smtp.transport.ts)
- [mailgun.transport.ts](../../../../../attendee-ems-back/src/modules/email/transport/mailgun.transport.ts)
- [queue-names.ts](../../../../../attendee-ems-back/src/infra/queue/queue-names.ts)

## Règles métier du fallback

### Emails éligibles

Le fallback SMTP est réservé aux emails transactionnels indispensables :

- confirmation/approbation d'inscription ;
- billet/QR prêt ;
- passage waitlist vers confirmé ;
- éventuellement réinitialisation de mot de passe si le support en a besoin.

Il est interdit par défaut pour :

- warm-up C3 ;
- newsletters et campagnes ;
- exports ou notifications admin non urgentes ;
- resend massif ;
- messages destinés à une adresse déjà bounce/complaint/blocked.

Chaque job porte donc une `messageClass` (`critical_transactional`, `transactional`, `bulk`) et une
`fallbackEligible` calculée côté serveur, jamais fournie librement par le client HTTP.

### Une erreur ne signifie pas toujours « envoyer ailleurs »

| Résultat primaire Mailgun | Politique C4 |
| --- | --- |
| 2xx + message id | Succès primaire, jamais de fallback |
| 429 avec `Retry-After` | Conserver dans la file primaire et ralentir ; pas de bascule immédiate |
| 5xx/connexion refusée avant envoi | Retry primaire borné, puis éligible au fallback si circuit ouvert |
| Timeout après émission possible | État `primary_unknown`; réconcilier avant toute bascule pour éviter un doublon |
| 401/403/config invalide | Alerte P0 ; bascule seulement après smoke SMTP et décision/mode autorisé |
| Destinataire invalide, bounce, complaint, blocked | Échec permanent ; **jamais** de fallback |
| Erreur de template/pièce jointe | Corriger la donnée ; changer de provider ne sert à rien |

## Architecture cible

```txt
email.send (orchestrateur, clé métier idempotente)
-> tentative Mailgun primaire
   -> accepted : attendre les webhooks Mailgun
   -> transient certain : retries bornés / circuit breaker
   -> état ambigu : primary_unknown + réconciliation/alerte
   -> fallback éligible : handoff atomique
-> email.send.smtp-fallback (lane séparée, concurrence et quota faibles)
-> SMTP accepted : statut smtp_accepted, jamais delivered sans preuve aval
```

### 1. Ledger durable et anti-doublon inter-provider

Ajouter un état durable par livraison, ou enrichir un modèle adapté :

```txt
delivery_key
message_class
state
primary_provider / primary_message_id / primary_attempts
fallback_provider / fallback_message_id / fallback_attempts
fallback_reason
created_at / accepted_at / delivered_at / failed_at
```

États minimum :

```txt
queued
primary_sending
primary_accepted
primary_unknown
fallback_eligible
fallback_sending
fallback_accepted
delivered
failed_permanent
```

Le passage `fallback_eligible -> fallback_sending` doit être atomique avec une contrainte unique sur
`delivery_key`. Un lock Redis seul ne suffit pas comme vérité durable. La clé métier C2 existante
reste la racine d'idempotence.

Ajouter une clé de corrélation Attendee au message Mailgun permet de rechercher un envoi ambigu
avant handoff. Si aucune réconciliation provider fiable n'est disponible, le timeout ambigu reste
en attente opérateur plutôt que de risquer un double envoi.

### 2. Lane SMTP séparée et bornée

- Queue dédiée `email.send.smtp-fallback` ou processor séparé à concurrence `1` au départ.
- Rate limit SMTP distinct de Mailgun.
- Compteur Redis atomique par minute et par journée Europe/Paris.
- `SMTP_FALLBACK_DAILY_CAP` configuré **en dessous** du quota contractuel confirmé.
- Réserver une part du budget aux billets/confirmations ; ne pas laisser les resends le consommer.
- Budget épuisé ou Redis indisponible : fail-closed, jobs conservés en file et alerte ; pas d'envoi
  incontrôlé.
- Timeouts Nodemailer explicites (`connectionTimeout`, `greetingTimeout`, `socketTimeout`) et pool
  borné.

Le compteur mesure les tentatives acceptées par le relais et garde une marge de sécurité. Il ne doit
pas être remis à zéro manuellement pour contourner une limite provider.

### 3. Circuit breaker et modes opératoires

Configuration recommandée :

```txt
EMAIL_FAILOVER_MODE=off|manual|auto
SMTP_FALLBACK_ENABLED=false|true
SMTP_FALLBACK_RATE_PER_SECOND=...
SMTP_FALLBACK_DAILY_CAP=...
SMTP_FALLBACK_RESERVED_CRITICAL=...
EMAIL_PRIMARY_FAILURE_THRESHOLD=...
EMAIL_PRIMARY_FAILURE_WINDOW_SECONDS=...
EMAIL_PRIMARY_COOLDOWN_SECONDS=...
```

- `off` : aucun fallback, jobs primaires restent en file.
- `manual` : le système qualifie les jobs, un opérateur ouvre la lane après smoke SMTP.
- `auto` : réservé aux erreurs transitoires certaines et aux classes critiques, dans les budgets.
- Retour primaire par état `half-open` et envoi canari ; jamais un retour massif instantané.
- Kill switch et pause séparés pour primaire et fallback.

Pour LFD, commencer en `manual`, observer un exercice contrôlé, puis n'activer `auto` que si les cas
d'erreur ambigus et les quotas ont une réponse vérifiée.

### 4. Délivrabilité et sémantique de statut

- Mailgun `accepted` puis webhooks `delivered/failed` restent la source la plus riche.
- SMTP `sendMail()` réussi signifie seulement « accepté par le relais SMTP ».
- La page/support affiche donc `envoyé au relais` ou `en cours`, pas `délivré`, pour le fallback.
- Les métriques sont séparées par provider et domaine destinataire ; ne pas mélanger SMTP et
  Mailgun dans les KPIs C3.
- Le `From`, SPF, DKIM et DMARC du chemin OVH doivent être testés avec le domaine réellement utilisé
  avant activation. Un fallback qui arrive en spam n'est pas un filet opérationnel.

## Lots d'implémentation

### C4.0 — Inventaire et garde-fous (~0,5 j)

- confirmer offre, quota, débit et politique anti-spam OVH ;
- smoke réel vers Gmail/Outlook/domaine pro, contrôle SPF/DKIM/DMARC ;
- timeouts/pool SMTP ;
- `messageClass`, éligibilité et configuration failover désactivée par défaut.

### C4.1 — Handoff durable et lane SMTP (~1–1,5 j)

- ledger de livraison/attempts et états ci-dessus ;
- classification des erreurs Mailgun ;
- queue/processor SMTP séparé, idempotence inter-provider ;
- rate limit minute/jour et budget réservé ;
- statut SMTP `accepted`, sans fausse promesse de délivrance.

### C4.2 — Circuit breaker, alertes et exercice (~0,5–1 j)

- modes `off/manual/auto`, cooldown et canari ;
- métriques/alertes quota restant, circuit, file, âge du plus vieux job, doublons ;
- runbook panne Mailgun et retour au primaire ;
- exercice staging avec panne Mailgun simulée puis quota SMTP volontairement atteint.

## Definition of Done C4

- [ ] SMTP et Mailgun restent deux transports distincts ; Mailgun demeure primaire.
- [ ] Le fallback ne concerne que les classes explicitement éligibles.
- [ ] Une erreur destinataire/template/bounce ne déclenche jamais SMTP.
- [ ] Un timeout Mailgun ambigu ne peut pas produire un double envoi automatique.
- [ ] Le handoff inter-provider est durable, atomique et idempotent par `delivery_key`.
- [ ] La lane SMTP a ses propres concurrence, débit, quota journalier et pause.
- [ ] Le quota réel OVH est confirmé ; la valeur applicative garde une marge documentée.
- [ ] Budget épuisé = jobs conservés + alerte, jamais dépassement silencieux.
- [ ] Les statuts `accepted` et `delivered` ne sont pas confondus.
- [ ] Tests : 2xx, 429/Retry-After, 5xx, timeout ambigu, 401, destinataire invalide, quota épuisé,
  restart worker, double enqueue, circuit half-open et retour Mailgun.
- [ ] Exercice staging : Mailgun KO simulé, un petit lot transactionnel passe par SMTP, aucun
  doublon, API stable, reprise primaire validée.

## Hors périmètre C4

- utiliser SMTP pour finir le warm-up C3 ou envoyer les 12 400 messages ;
- remplacer Mailgun comme ESP principal ;
- basculer sur toute erreur sans classification ;
- prétendre mesurer la délivrance SMTP sans événement aval ;
- gérer le fallback PDF/Puppeteer (chantier B2).
