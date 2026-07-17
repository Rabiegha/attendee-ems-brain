# C2.1 — Suivi orchestration billet PDF + email session + suivi minimal

## Statut

```txt
Statut: cadre
Avancement: 5 %
Estime: ~5-7 j
Owner: Rabie
Dependances: C2 pour l'envoi email, B0/B1 pour le PDF, BIL pour les decisions billet/QR/parcours
Inclus: pipeline PDF/email + page confirmation persistante + polling leger + telechargement billet
Hors perimetre: websocket temps reel, resend avance, correction email self-service, verification email obligatoire
```

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
- Preferer un QR stable par registration/attendee et un billet PDF versionne.
- Ajouter l'email billet session dans l'onglet Email de l'evenement, pas dans chaque session en V1.
- Garder les parametres session centres sur ouverture/capacite/waitlist.
- Mettre en place un worker PDF/ticket separe du process API.
- Mettre en place un worker email separe du process API comme cible C2.1, au plus tard avant les tests de volume realistes.
- Ajouter une page confirmation persistante avec polling leger.
- Afficher billet en preparation, billet pret, email en attente/envoye.
- Donner un bouton de telechargement billet quand le PDF est pret.
- Ne pas mettre la reservation de place elle-meme dans un worker asynchrone.

## Points ouverts

- Table exacte : `tickets`, `registration_tickets`, `ticket_versions`, ou autre nom.
- Outbox DB vs enqueue BullMQ directement apres commit.
- Piece jointe PDF obligatoire ou lien de secours prioritaire.
- Mapping minimal entre `email_events` Mailgun et l'etat billet visible support.
- Timing exact de separation du worker email du process API si l'arbitrage planning force une phase intermediaire.
- Packaging/deploiement des workers separes : process Nest distincts, services systemd/containers dedies, ou autre.
- Niveau de mock PDF/email pour les tests de charge.
- Texte exact des messages utilisateur et timeout d'attente avant conseil spam/support.

## Prochaine action

Demarrer apres un chemin PDF exploitable cote B :

```txt
1. choisir le modele ticket/version/status
2. ajouter job durable ticket.generate
3. brancher generation PDF
4. creer worker ticket/pdf separe
5. separer worker email du process API
6. creer endpoint/status public de confirmation
7. creer page confirmation + polling leger + telechargement billet
8. enqueue email.send avec idempotence
9. ajouter template/config email billet
```
