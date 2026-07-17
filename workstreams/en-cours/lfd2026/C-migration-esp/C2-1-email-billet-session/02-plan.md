# C2.1 — Plan orchestration billet PDF + email session + suivi minimal

## Objectif

Mettre en place une pipeline technique minimale et fiable pour orchestrer le billet PDF puis
l'email billet apres une inscription session, avec PDF genere par le chantier B/Gotenberg,
decisions produit alignees avec BIL, et une UX de suivi minimale, sans ralentir ni fragiliser
l'inscription.

Le flux cible C2.1 :

```txt
POST inscription session
-> transaction DB:
   - creer/mettre a jour registration
   - creer RegistrationSessionChoice
   - confirmer ou waitlister la place
   - creer/invalider un ticket/billet record
   - creer une demande durable de generation PDF
-> commit
-> worker PDF genere ou regenere le billet
-> stocke le PDF
-> met ticket status ready
-> page confirmation affiche le billet quand il est pret
-> enqueue email.send avec PDF ou lien
-> worker email C2 envoie via Mailgun
```

Le PDF fait partie de la pipeline metier, mais pas du temps de reponse utilisateur.

## Dependances

### Dependance B

C2.1 depend de B pour avoir au minimum :

- un endpoint/service de generation PDF utilisable par le back
- un stockage du PDF ou une reference stable
- un format de billet acceptable pour email
- une decision piece jointe vs lien de secours

B n'a pas besoin d'etre totalement finalise pour demarrer C2.1, mais il faut au moins un chemin
PDF utilisable.

### Dependance BIL

C2.1 doit rester aligne avec BIL pour :

- decision QR stable par registration vs QR par session
- contenu produit du billet
- logique de version de billet
- parcours utilisateur cible apres inscription
- contraintes futures de backoffice client

### Dependance C2

C2.1 reutilise :

- BullMQ `email.send`
- `EmailService.sendEmail`
- `sendFromCustomTemplate`
- Mailgun transport API EU
- webhooks Mailgun
- `email_events`
- throttle dynamique Redis

Il ne faut pas creer une deuxieme infrastructure email.

## Modele technique recommande

Ajouter une couche billet, par exemple :

```txt
registrations
registration_session_choices
tickets / registration_tickets
ticket_versions
ticket_generation_jobs ou outbox_events
email_deliveries ou lien metier vers email_events
```

Minimum viable C2.1 :

- un record ticket par registration ou registration/session selon la decision QR
- un `ticket_status`
- une version de billet
- une reference PDF stockee
- un job durable de generation PDF
- un email billet idempotent

Etats proposes :

```txt
ticket_status:
- pending
- generating
- ready
- failed
- outdated

email_status:
- not_queued
- queued
- sending
- sent
- delivered
- bounced
- failed
```

Ces statuts peuvent etre simples au debut. C2.1 doit inclure une page de suivi minimale basee
sur ces statuts. Le websocket temps reel avance reste hors perimetre.

## UX de suivi minimale incluse

C2.1 doit livrer une page de confirmation persistante, reload-safe, accessible par token non
devinable.

Flux UX cible :

```txt
POST inscription session
-> 303 redirect vers /registration-confirmation/:token
-> page confirmation:
   - inscription confirmee
   - billet en preparation
   - email en attente
-> polling leger vers endpoint status
-> quand PDF pret:
   - bouton telecharger le billet
   - message "vous allez aussi le recevoir par email"
-> quand email envoye:
   - message email envoye a xxx@example.com
```

Comportement reload :

```txt
Reload = GET page de confirmation
Pas de nouveau POST inscription
Pas de regeneration PDF non idempotente
Pas de renvoi email automatique non controle
```

Endpoint status attendu :

```txt
GET /registration-confirmation/:token/status
```

Exemple de payload :

```json
{
  "registration": "confirmed",
  "ticket": {
    "status": "generating",
    "version": 3,
    "downloadUrl": null
  },
  "email": {
    "status": "queued"
  }
}
```

Quand le billet est pret :

```json
{
  "registration": "confirmed",
  "ticket": {
    "status": "ready",
    "version": 3,
    "downloadUrl": "/tickets/download/..."
  },
  "email": {
    "status": "sent"
  }
}
```

Polling recommande :

```txt
toutes les 2s pendant 30s
puis toutes les 10s
message de patience apres quelques minutes
```

Message support minimal :

```txt
Si vous ne recevez pas l'email dans les 10 minutes, verifiez vos spams.
Votre billet reste telechargeable depuis cette page.
```

## QR code et version de billet

Decision recommandee pour LFD2026 :

```txt
Un QR stable par registration/attendee.
Un PDF regenerable qui reflete les sessions actuelles.
Une version de billet.
```

Pourquoi :

- moins de billets et de mails
- moins de confusion pour l'utilisateur
- le QR reste valide si les sessions changent
- le controle d'acces verifie dynamiquement les droits de l'attendee a la session

Flux avec QR stable :

```txt
Ajout/retrait/changement de session
-> ticket_version++
-> ancien PDF devient outdated
-> job generate_ticket_pdf(registration_id, ticket_version)
-> quand pret, PDF vN devient current
-> email billet envoye si necessaire
```

Alternative non retenue en V1 :

```txt
Un QR/PDF par session.
```

Cette option donne un billet tres explicite par session, mais augmente fortement le nombre de
PDF, emails, retries et cas de support.

## Integration dans l'inscription session

A la fin de l'inscription session, ne pas generer le PDF directement.

Faire plutot :

```txt
RegistrationSessionChoice creee
-> demande generation PDF idempotente
```

La transaction garantit l'inscription. La queue garantit que PDF/email arriveront ensuite.

Important :

- l'inscription ne doit pas attendre Gotenberg
- l'inscription ne doit pas attendre Mailgun
- un reload front ne doit jamais relancer un POST ni doubler les jobs
- les jobs doivent etre idempotents

Cles idempotentes possibles :

```txt
ticket:<registrationId>:v<ticketVersion>:generate
ticket:<registrationId>:v<ticketVersion>:email
reg:<registrationId>:session:<sessionId>:ticket-email
```

## Configuration UX admin

Ne pas mettre toute la configuration email billet dans les parametres de chaque session en V1.

Recommandation :

```txt
Onglet Email de l'evenement
- Email confirmation inscription event
- Email approbation
- Email refus
- Email billet session
  - active oui/non
  - template
  - inclure PDF en piece jointe oui/non
  - inclure lien de telechargement oui/non
```

Dans les parametres session, garder seulement les controles metier :

```txt
- inscription publique active
- ouverture/fermeture
- date d'ouverture
- capacite
- waitlist
```

Override par session a garder pour plus tard :

```txt
Session X utilise un template specifique.
```

Ne pas commencer par cet override : sinon on cree trop vite une matrice lourde
`event emails + session emails + overrides + activation conditionnelle`.

## Templates email

C2.1 doit prevoir une petite refonte des templates email pour les billets :

- nouveau type/logique "email billet session"
- variables billet :
  - `firstName`
  - `lastName`
  - `email`
  - `eventName`
  - `sessionName`
  - `sessionStartTime`
  - `sessionLocation`
  - `ticketDownloadUrl`
  - `registrationId`
  - `ticketVersion`
- support PDF en piece jointe
- lien de secours vers PDF

Le template exact depend du chantier B, car il faut savoir comment le PDF est genere et stocke.

## Workers separes

C2 indique que le worker email tourne encore dans le process API. Pour LFD2026, il faut envisager
de separer les workers qui ne sont pas le chemin critique HTTP.

Recommandation :

1. **Worker PDF/ticket separe : requis pour C2.1.**
   - Gotenberg/Cloud Run + rendu PDF = latence et CPU/reseau.
   - Ce travail ne doit pas partager le cycle de vie du process HTTP.
   - On peut scaler le worker PDF independamment.
   - Un incident PDF ne doit pas degrader l'API d'inscription.

2. **Worker email separe : cible C2.1, a faire avant les tests de volume realistes.**
   - L'infra BullMQ existe deja.
   - Separation possible en reutilisant les memes modules NestJS et Redis.
   - Benefice : l'API HTTP reste concentree sur inscriptions/check-in.
   - Le throttle Mailgun reste pilotable a chaud.
   - Un ralentissement Mailgun ou un backlog email ne doit pas consommer les ressources du process HTTP.

3. **Workers pour inscriptions : non pour la reservation elle-meme.**
   - L'inscription/capacite doit rester synchrone et transactionnelle pour donner une reponse fiable.
   - On peut externaliser les effets secondaires apres commit : PDF, email, scoring, analytics, notifications admin.
   - Ne pas mettre la reservation de place elle-meme dans une queue si l'utilisateur attend une reponse immediate sur sa place.

Mode cible raisonnable :

```txt
api-web:
  - HTTP inscription/session/check-in
  - transactions courtes
  - enqueue durable des effets secondaires

worker-email:
  - BullMQ email.send
  - throttle Mailgun
  - retry API Mailgun

worker-ticket:
  - generate_ticket_pdf
  - appel Gotenberg
  - stockage PDF
  - enqueue email billet

worker-low-priority:
  - scoring
  - affinities
  - analytics
  - admin notifications si besoin
```

## Hors perimetre C2.1

Les sujets suivants sont importants, mais doivent etre traites dans un chantier suivant :

- websocket de progression ticket/email
- bouton resend avance
- correction email par utilisateur
- message "si pas recu dans 10 minutes"
- mapping complet delivrabilite Mailgun vers interface support
- verification email par code avant inscription

C2.1 doit livrer le suivi minimal. Le chantier suivant pourra enrichir en temps reel, support,
resend/correction email et verification.

## Tests et montee en charge

Une fois B + C2.1 suffisamment en place :

```txt
test charge inscription seule
test charge inscription + PDF mock
test charge inscription + PDF Gotenberg
test charge inscription + email queue mock/sandbox
test charge complet inscription + PDF + email + retries
```

Le but est de mesurer le debit reel final, puis seulement ensuite decider s'il faut appliquer
d'autres leviers.

## Checklist C2.1

- [ ] Valider decision QR stable par registration vs QR par session.
- [ ] Ajouter modele ticket/billet + version + status.
- [ ] Ajouter queue/job durable `ticket.generate`.
- [ ] Brancher generation PDF issue de B.
- [ ] Creer un worker ticket/PDF separe du process API.
- [ ] Separer le worker email du process API avant les tests de volume realistes, sauf arbitrage explicite.
- [ ] Stocker PDF + reference de telechargement.
- [ ] Ajouter token/page confirmation persistante.
- [ ] Ajouter endpoint status confirmation.
- [ ] Ajouter polling leger cote front.
- [ ] Afficher billet en preparation / billet pret / email en attente-envoye.
- [ ] Ajouter bouton telechargement billet quand le PDF est pret.
- [ ] Ajouter email billet session dans la configuration event.
- [ ] Ajouter support template email billet.
- [ ] Enqueue `email.send` avec idempotence et PDF/lien.
- [ ] Prevoir mapping minimal email status.
- [ ] Documenter les limites : pas encore websocket, resend avance, correction email.
- [ ] Documenter le mode de deploiement des workers separes.
