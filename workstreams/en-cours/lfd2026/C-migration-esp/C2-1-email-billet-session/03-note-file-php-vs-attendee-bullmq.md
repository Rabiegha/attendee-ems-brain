# 03 — Note decision — file PHP locale vs file Attendee/BullMQ

> Date : 2026-07-17  
> Sujet : justifier pourquoi la file locale `back-office-event` (`data/queue.jsonl` + `worker.php`) peut aider temporairement, mais ne doit pas devenir le choix cible pour PDF + email billet.

## Contexte

Le formulaire `back-office-event` utilise aujourd'hui ce flux :

```txt
front PHP
-> POST register.php
-> POST Attendee /api/public/events/sessions/:sessionToken/register
-> Attendee reserve la place et cree/reutilise la registration event + session choice
-> register.php depose un job local dans data/queue.jsonl
-> worker.php est cense generer le billet PDF puis envoyer l'email
```

Ce choix a une qualite : il garde la generation PDF/email hors de la requete d'inscription.
Mais il place l'orchestration durable du billet hors d'Attendee.

## Decision recommandee

Pour LFD2026, la cible doit etre :

```txt
POST /api/public/events/sessions/:sessionToken/register
-> transaction courte d'inscription/reservation
-> commit
-> enqueue durable cote Attendee :
   - ticket.generate
   - email.send
-> worker ticket/PDF
-> worker email Mailgun
```

Le PHP peut rester un adaptateur de formulaire, mais ne doit pas etre proprietaire de la file metier
billet/email.

## Regle d'architecture cible : un endpoint declenche, les workers executent

Pour l'inscription session LFD, un seul endpoint public doit declencher le workflow metier :

```txt
POST /api/public/events/sessions/:sessionToken/register
```

Ce endpoint fait uniquement le travail synchrone necessaire pour donner une reponse fiable au
participant :

```txt
1. verifier que l'event et la session sont ouverts
2. reserver la place sans survente
3. creer ou reutiliser la registration event
4. creer le choix de session
5. creer/enqueue les jobs durables PDF/email
6. repondre rapidement au front
```

Ce endpoint ne doit pas faire directement :

```txt
- generation PDF
- appel bloquant a Gotenberg
- envoi bloquant Mailgun
```

Le flux cible devient :

```txt
register endpoint
-> commit DB
-> job ticket.generate
-> worker ticket genere le PDF
-> stockage PDF + ticket_status=ready
-> enqueue email.send
-> worker email envoie via Mailgun
-> webhooks Mailgun mettent a jour email_events
```

Le front recoit donc une reponse rapide du type "inscription confirmee, billet en preparation",
puis la page de confirmation/polling affiche ensuite "billet pret" et "email envoye".

## Pourquoi la file PHP locale n'est pas un bon choix a terme

- **Deux sources de verite** : Attendee possede la registration, la session, l'email, les statuts et les idempotency keys ; la file PHP duplique une partie de ce contexte.
- **Durabilite faible** : un fichier `queue.jsonl` local depend du disque, du verrou fichier, du cron/worker manuel et n'a pas de dead-letter queue robuste.
- **Observabilite limitee** : pas de Bull Board, pas de metriques queue standard, pas de retry/backoff centralise, pas de statut billet/email visible facilement depuis Attendee.
- **Idempotence fragile** : Attendee peut garantir des cles comme `ticket:<registrationId>:v<version>:generate` ou `reg:<registrationId>:session:<sessionId>:ticket-email`; le PHP travaille avec un `ticket_id` local moins relie a la verite metier.
- **Support utilisateur plus dur** : si un participant appelle, l'equipe doit recouper Attendee + fichiers PHP pour savoir si le billet est genere, envoye, bloque ou perdu.
- **Scalabilite faible** : plusieurs instances PHP ou plusieurs workers fichier augmentent les risques de concurrence, doublons, jobs perdus ou ordre mal gere.
- **Mauvais endroit pour C2/Mailgun** : C2 a deja BullMQ, `email.send`, throttle Mailgun, webhooks et `email_events`. Refaire une deuxieme infra email cote PHP complexifie inutilement.
- **Migration future plus couteuse** : plus on branche PDF/email autour du fichier PHP, plus il faudra detricoter ensuite pour revenir vers Attendee.

## Pourquoi Attendee/BullMQ est le bon choix cible

- Attendee possede la **verite DB** : `registration_id`, `session_id`, statut, email, ticket version.
- BullMQ apporte **retry**, **backoff**, **jobId idempotent**, **dead-letter/logs**, monitoring et separation des workers.
- C2 fournit deja l'infra email : Mailgun, throttle, webhooks, `email_events`.
- B/Gotenberg peut etre appele par un worker ticket sans bloquer l'API HTTP.
- La page de confirmation C2.1 peut lire un statut unique dans Attendee : inscription, billet, email.

## Position court terme

La file PHP locale peut rester acceptable uniquement comme rustine courte duree si :

- elle ne bloque pas L9b/L9a/L9.1 ;
- elle ne devient pas la source de verite du billet ;
- elle ne porte pas la logique Mailgun definitive ;
- elle est remplacee par `ticket.generate` + `email.send` cote Attendee des que C2.1 demarre.

## Regle a garder

PDF et email doivent rester **asynchrones**, mais leur orchestration durable doit vivre cote
Attendee, pas dans un fichier local du formulaire.
