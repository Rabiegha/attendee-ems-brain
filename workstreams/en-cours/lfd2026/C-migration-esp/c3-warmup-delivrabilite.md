# C3 — Warm-up Mailgun & délivrabilité opérationnelle

> Créé le 15/07/2026 à partir de la bascule C1/C2.
> C1 = domaine/DNS/setup Mailgun. C2 = intégration applicative.
> C3 = exploitation réelle : warm-up, métriques, webhooks, qualité inbox.

## Objectif

Amener `mail.attendee.fr` à une réputation suffisante pour tenir le pic LFD 2026 sans bloquer
les inscriptions, tout en surveillant les signaux mailbox providers.

## État de départ

✅ Fait :

- Plan Mailgun acheté.
- Domaine EU `mail.attendee.fr` créé et vérifié.
- DNS OVH publiés : SPF, DKIM, DMARC, MX, CNAME tracking.
- Clé API Mailgun rotatée et disponible.
- Variables d'env Mailgun posées en staging et prod.
- Webhooks Mailgun créés pour staging et prod.
- Envoi testé en staging.
- Envoi testé en prod.
- Base warm-up disponible : inscrits newsletter du magasin Channel Scope.

## À faire

- [ ] Valider que les événements webhooks arrivent bien en base `email_events`.
- [ ] Vérifier les stats `/email/events/stats` après premiers envois.
- [ ] Définir le débit initial côté throttle email (`1 email/s` recommandé J1).
- [ ] Créer/choisir le mécanisme d'envoi warm-up : script interne passant par la queue Attendee,
      pas une boucle Mailgun brute.
- [ ] Préparer le contenu warm-up Channel Scope avec opt-out clair.
- [ ] Nettoyer la base newsletter avant envoi : doublons, emails invalides, adresses rôle si possible.
- [ ] Segmenter par fournisseur destinataire : Gmail, Outlook/Hotmail, Yahoo, domaines pro.
- [ ] Envoyer J1 en petits lots, observer inbox/spam, bounce, complaint, deferred.
- [ ] Documenter les résultats J1 dans ce fichier.
- [ ] Ajuster J2/J3 selon signaux réels.
- [ ] Écrire le runbook jour J : pause, baisse débit, contact support Mailgun, fallback billet hors email.

## Critères de succès

- Bounce hard très faible.
- Aucune plainte spam.
- Gmail/Outlook/Yahoo livrent en inbox ou au minimum sans blocage massif.
- Webhooks `accepted`, `delivered`, `temporary failure`, `permanent failure`, `spam complaints`
  visibles côté Mailgun et côté Attendee.
- Queue email stable, débit modifiable sans restart.

## Risques spécifiques

- Utiliser une base newsletter non alignée avec le domaine Attendee peut créer des plaintes.
- Envoyer trop vite sur Gmail/Outlook au J1 peut déclencher des ralentissements.
- Un deuxième domaine d'envoi non warmé n'aide pas le jour J ; il peut même fragmenter la réputation.
