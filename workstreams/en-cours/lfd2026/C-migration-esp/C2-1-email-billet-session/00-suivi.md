# C2.1 — Suivi orchestration billet PDF + email session + suivi minimal

## Statut

```txt
Statut: MVP livré (email sans PDF) — reste PDF + page confirmation
Avancement: 25 %
Estime: ~3-4 j restants (PDF + page confirmation)
Owner: Rabie
Dependances: C2 pour l'envoi email (FAIT), B0/B1 pour le PDF (bloquant pour la suite), BIL pour les decisions billet/QR/parcours
Inclus: pipeline PDF/email + page confirmation persistante + polling leger + telechargement billet
Hors perimetre: websocket temps reel, resend avance, correction email self-service, verification email obligatoire
```

## Journal des changements

### 2026-07-20 — MVP email session sans PDF (livré sur staging)

**Contexte :** deadline serrée, B0 non terminé. Décision de livrer un MVP email sans PDF
pour pouvoir présenter un système d'envoi fonctionnel.

**Prérequis résolus avant le MVP :**

1. **Staging ne envoyait pas de mails** — `.env.staging` pointait sur `SMTP_HOST=mailpit`
   (guardrail load test par conception). Basculé sur `EMAIL_TRANSPORT=mailgun` avec les
   mêmes credentials que la prod. Rebuild container staging.

2. **Bug `!isAdminSource` dans `registrations.service.ts`** — la condition était inversée :
   l'email était sauté pour les inscriptions `public_form` et envoyé pour les sources admin.
   Corrigé : `if (!isAdminSource)` → `if (isAdminSource)`.
   Ce bug existait aussi en prod, mais les emails prod "fonctionnaient" via `approve-with-email`
   (bouton admin manuel), un chemin entièrement séparé qui bypass la condition.

**Ce qui a été codé (commit `86806d9` sur `staging`) :**

Dans `src/modules/public/public.service.ts`, fonction `registerToSession` :
- Ajout de `emailSettings: { include: { approvalTemplate: true } }` dans le include Prisma
  de la session (pour charger le template dans la même transaction)
- Retour d'un `_emailContext` interne depuis la transaction (données nécessaires à l'email,
  strippé du retour API public)
- `setImmediate` post-transaction : envoi fire-and-forget de l'approval template via
  `emailService.sendFromCustomTemplate`
- Clé d'idempotence : `reg:<registrationId>:session:<sessionId>:approval`
- Variable `sessionName` exposée au template pour personnalisation

**Conditions requises côté config back-office pour que l'email parte :**
- `approval_enabled = true` dans les email settings de l'event
- Un approval template lié (`approval_template_id` non NULL avec `template_data`)
- Inscription `confirmed` (pas `waitlisted`) et première inscription (`already_registered=false`)

**Ce que le MVP ne fait PAS :**
- Pas de PDF en pièce jointe (dépend de B0/Gotenberg)
- Pas de page de confirmation persistante
- Pas d'email sur waitlist → confirmé
- Pas d'email sur désinscription

## Pourquoi ce chantier existe

C2 a livre l'infrastructure email robuste : Mailgun, BullMQ, webhooks, throttle, idempotence.
B travaille la generation PDF via Gotenberg/Cloud Run.

C2.1 fait le pont entre C2, B et BIL :

```txt
inscription session
-> generation billet PDF asynchrone
-> page de suivi minimale cote utilisateur
-> email billet via l'infra C2
```

Ce chantier ne remplace pas B. Il inclut maintenant l'UX minimale necessaire pour que le
parcours soit exploitable sans attendre un chantier UX complet.

## Documents lies

- [01-etat-des-lieux.md](./01-etat-des-lieux.md)
- [02-plan.md](./02-plan.md)
- [C2 recap](../c2-recap-2026-07-14.md)
- [C2 migration technique](../mailgun-migration-technique.md)
- [B email -> billet PDF](../../B-email-billet-pdf/README.md)
- [BIL billetterie](../../BIL-billetterie/README.md)
- [Suivi chantiers LFD2026](../../03-suivi-chantiers.md)

## Decisions actuelles

- Reutiliser `EmailService`, `EmailQueueService`, BullMQ `email.send`, Mailgun et webhooks C2.
- Ne pas creer un deuxieme systeme email.
- Ne pas generer le PDF dans la requete HTTP d'inscription.
- Ajouter une demande durable de generation PDF apres inscription session.
- Utiliser un QR stable par registration/attendee et le `Badge` deja unique par `registration_id`.
- Ne pas ajouter de `ticket_id` ou `ticket_version` au MVP : le PDF ne liste pas les sessions et
  le controle des droits session reste dynamique au scan.
- Ajouter l'email billet session dans l'onglet Email de l'evenement, pas dans chaque session en V1.
- Garder les parametres session centres sur ouverture/capacite/waitlist.
- Mettre en place un worker PDF separe du process API (detail et garde-fous dans B2).
- Mettre en place un worker email separe du process API comme cible C2.1, au plus tard avant les tests de volume realistes.
- Ajouter une page confirmation persistante avec polling leger.
- Afficher billet en preparation, billet pret, email en attente/envoye.
- Donner un bouton de telechargement billet quand le PDF est pret.
- Ne pas mettre la reservation de place elle-meme dans un worker asynchrone.

## Points ouverts

- Outbox DB vs enqueue BullMQ directement apres commit.
- Piece jointe PDF obligatoire ou lien de secours prioritaire.
- Mapping minimal entre `email_events` Mailgun et l'etat billet visible support.
- Timing exact de separation du worker email du process API si l'arbitrage planning force une phase intermediaire.
- Packaging/deploiement des workers separes : process Nest distincts, services systemd/containers dedies, ou autre.
- Niveau de mock PDF/email pour les tests de charge.
- Texte exact des messages utilisateur et timeout d'attente avant conseil spam/support.

Voir aussi : [B2 — fallback Puppeteer resilient et asynchrone](../../B-email-billet-pdf/B2-fallback-puppeteer-resilient.md).

## Prochaine action

MVP livré. La suite dépend de la validation/merge B1, puis partage son worker avec B2 :

```txt
FAIT (MVP 2026-07-20) :
  ✓ email approval template envoyé après inscription session confirmée
  ✓ idempotence via clé reg:<registrationId>:session:<sessionId>:approval
  ✓ variable sessionName disponible dans le template

RESTE (après B1) :
  1. PDF généré après inscription session (worker Gotenberg)
  2. PDF attaché à l'email (ou lien R2 signé de secours)
  3. Page confirmation persistante (/registration-confirmation/:token)
  4. Endpoint status GET .../status avec polling léger
  5. Email sur waitlist → confirmé
  6. Worker email séparé du process API (avant tests de volume)
```
