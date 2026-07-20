# C2.1 — Etat des lieux orchestration billet PDF + email session

> **Note de mise à jour 20/07/2026 :** ce document décrivait l'état avant le MVP. L'email
> d'approbation est maintenant déclenché après l'inscription session confirmée. Le PDF durable,
> son statut et la page de suivi restent à faire. Le modèle `Badge`, unique par `registration_id`,
> est réutilisé ; aucun nouveau `ticket_id`/`ticket_version` n'est requis pour le MVP.

## Resume

C2 a pose une bonne base pour l'envoi email transactionnel : transport Mailgun, queue BullMQ,
retry, throttle dynamique, idempotence et webhooks de delivrabilite. B porte la generation PDF,
et BIL porte les decisions produit billet/QR/parcours. C2.1 doit orchestrer ces briques plutot
que creer un deuxieme systeme email ou melanger PDF/email dans la requete d'inscription.

Avant le MVP du 20/07, l'inscription publique a une session ne declenchait ni email ni PDF. Elle
cree ou reutilise toujours la registration event, ajoute le choix session et diffuse les compteurs
live ; l'email d'approbation est maintenant enfile apres commit, mais le PDF/statut reste absent.

## Ce que C2 permet deja

Les emails passent par `EmailService`. Si `EMAIL_QUEUE_ENABLED=true`, l'appel ne part pas
directement chez Mailgun : il est enfile dans BullMQ `email.send`.

Reference code :

- [email.service.ts](/Users/apple/Projects/ems-app/attendee-ems-back/src/modules/email/email.service.ts:87)

La queue email gere les pieces jointes en base64 et l'idempotence via `idempotencyKey`,
transformee en `jobId` BullMQ deterministe.

Reference code :

- [email-queue.service.ts](/Users/apple/Projects/ems-app/attendee-ems-back/src/modules/email/queue/email-queue.service.ts:26)

Le worker email gere :

- retry BullMQ
- throttle dynamique Redis
- appel au transport actif `smtp` ou `mailgun`
- trace `accepted` / `api_error` dans `email_events`
- webhooks Mailgun pour les evenements aval de delivrabilite

Reference code :

- [email-send.processor.ts](/Users/apple/Projects/ems-app/attendee-ems-back/src/modules/email/queue/email-send.processor.ts:10)

C2 a donc bien prevu Mailgun, webhooks, throttle et idempotence. Limite connue : selon le
recap C2, le worker BullMQ tourne encore dans le process API, pas dans un process separe.

Reference contexte :

- [c2-recap-2026-07-14.md](../c2-recap-2026-07-14.md)
- [mailgun-migration-technique.md](../mailgun-migration-technique.md)

## Flux confirmation event existant

Pour les confirmations classiques d'inscription event, le flux existe deja. Apres inscription
publique event, le code regarde :

- `approval_enabled`
- `confirmation_enabled`
- `approvalTemplate`
- `confirmationTemplate`

Puis il appelle `sendFromCustomTemplate` avec une cle idempotente, par exemple :

```txt
reg:<registrationId>:confirmation
reg:<registrationId>:approval
```

Reference code :

- [public.service.ts](/Users/apple/Projects/ems-app/attendee-ems-back/src/modules/public/public.service.ts:910)

Point a noter : ce flux utilise encore `setImmediate(...)` pour declencher l'envoi apres la
creation. Avec BullMQ derriere, l'ESP ne bloque pas l'inscription, mais il reste une petite
fenetre de perte possible si l'API crash apres le commit DB et avant l'enqueue.

## Flux inscription session audite avant le MVP

Aujourd'hui, `registerToSession` :

1. resout la session par `public_token`
2. verrouille la ligne session avec `FOR UPDATE`
3. verifie ouverture/capacite/waitlist
4. upsert l'attendee
5. cree ou reutilise la registration event
6. cree `RegistrationSessionChoice`
7. emet les stats live WebSocket de la session
8. retourne au front

Reference code :

- [public.service.ts](/Users/apple/Projects/ems-app/attendee-ems-back/src/modules/public/public.service.ts:357)

Ce qui manque après le MVP du 20/07 :

- pas de generation PDF apres inscription session
- pas encore d'usage coherent de `Badge.status/pdf_url/error_message` pour ce flux asynchrone
- pas de `email_status` lie au billet
- pas de page de suivi persistante
- pas de resend/correction email
- pas de mapping direct Mailgun event -> registration/badge

## Email verification

`require_email_verification` existe deja dans les settings et dans l'UI, mais l'analyse du flux
public ne montre pas encore une vraie verification email par code dans le parcours public.
Pour l'instant, il faut donc le traiter comme un flag/config existant, pas comme un systeme
complet de verification pre-inscription.

Decision actuelle recommandee :

```txt
Ne pas bloquer l'inscription session sur une verification email obligatoire.
Afficher le billet sur le web dans C2.1 via une page de suivi minimale.
Permettre resend/correction email plus tard, apres C2.1.
```

## Conclusion

Oui, C2.1 doit reutiliser C2 et dependre explicitement de B/BIL :

```txt
EmailService
EmailQueueService
templates email
Mailgun transport
Mailgun webhooks
throttle dynamique
idempotencyKey
B/Gotenberg pour le PDF
BIL pour les decisions QR/billet/parcours
```

Mais C2.1 doit ajouter une couche metier au-dessus :

```txt
inscription session confirmee
-> ticket/billet pending ou outdated
-> job durable generate_ticket_pdf
-> PDF pret
-> email.send avec PDF ou lien
```

Le point critique n'est pas l'envoi email pur : il est deja bien amorce par C2. Le vrai travail
C2.1 est le sequencement fiable entre inscription session, generation PDF et email billet.
