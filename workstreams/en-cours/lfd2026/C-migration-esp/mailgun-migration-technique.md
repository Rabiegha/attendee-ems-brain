# MAILGUN — Migration technique Attendee (Chantier C2)

> **Objet :** intégration applicative de **Mailgun (région EU, IP partagée, API HTTP)** dans le
> back Attendee (NestJS), en remplacement du **nodemailer SMTP direct** actuel.
> **Décision ESP :** [esp-choix-mailgun.md](./esp-choix-mailgun.md) · **Warm-up / DNS :** [MAILGUN_WARMUP_PLAN.md](./MAILGUN_WARMUP_PLAN.md)
> **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-C2)
>
> ✅ **Avançable en parallèle de C1** (n'a **pas** besoin de l'accès OVH — tests via domaine sandbox
> Mailgun ou Mailpit en staging). Seule la **bascule finale** dépend du domaine `mail.attendee.fr` vérifié.

---

## 1. État actuel (à migrer)

- Envoi = **`nodemailer` SMTP direct** (pas d'ESP), déclenché en `setImmediate(...)` fire-and-forget
  in-process (`src/modules/public/public.service.ts`) → **non durable, sans retry, sans idempotence**.
- Cf. [diagnostic email/billet/wallet](../B-email-billet-pdf/email-billet-wallet.md) et
  [BACKLOG-TECH § Redis/BullMQ async email](../../../../backlog/a-faire/BACKLOG-TECH.md).

## 2. Architecture cible

```
Front inscription → API NestJS → DB
                         │
                         └─► Queue BullMQ (job email.send, jobId déterministe = idempotence)
                                   │
                                   └─► Worker email NestJS ──API HTTP──► Mailgun EU
                                                                            │
                       Webhooks (accepted/delivered/deferred/bounce/spam) ◄┘
                                                   │
                                          Table de suivi des événements → back-office
```

**Principes (issus du rapport de décision) :**

- **Ne jamais envoyer depuis la requête HTTP d'inscription** → enqueue puis worker.
- **API HTTP** privilégiée au SMTP (plus fiable à l'échelle chez Mailgun).
- **Idempotence** via `jobId` déterministe (anti double-submit).
- **Webhooks** pour le suivi (pas de polling).
- **Débit worker pilotable à chaud** (Redis/DB), sans restart API.
- **Monitoring par domaine destinataire** pour distinguer Gmail, Outlook/Hotmail, Yahoo et domaines pro.
- **Mailgun gère les retries de livraison** côté `deferred`, mais Attendee doit observer les events
  et adapter son débit.

### Décision 14/07 — Option B : throttle dynamique hors process

Ne pas se limiter à un `limiter` BullMQ statique lu au démarrage du worker. Ce serait simple, mais
changer le débit pendant l'event imposerait un redémarrage du worker, voire une intervention ops au
mauvais moment.

Option retenue : **throttle applicatif dynamique**.

```txt
Redis / DB
  email:rate_limit_per_second
  email:paused
  email:rate_limit:gmail
  email:rate_limit:outlook
  email:rate_limit:yahoo

Worker email
  lit le réglage courant avant d'envoyer
  attend son slot d'envoi
  appelle Mailgun
  enregistre accepted / erreur API

Webhooks Mailgun
  enregistrent delivered / deferred / bounce / complained / complaint / failed
  alimentent monitoring + décisions de ralentissement
```

Débit cible LFD à valider avec Mailgun :

```txt
Besoin pic : 3 000 confirmations email en 3-4 min
Débit submit : ~13-17 emails/s
Plafond initial demandé : 20 emails/s pendant 3-5 min
```

Règles de contrôle :

- si `deferred` augmente fortement sur Gmail → ralentir Gmail si possible, sinon débit global ;
- si `deferred` augmente partout → pause courte / backoff global ;
- si `bounce` hard → pas de retry applicatif ;
- si `complained` / spam → alerte critique, ne pas monter le débit ;
- si Mailgun renvoie une erreur de rate limit/API temporaire → backoff worker immédiat ;
- les jobs restent en queue, l'inscription ne doit pas échouer à cause de l'email.

## 3. Tâches backend

| Tâche                | Détail                                                                                                                                          |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Config région EU** | client Mailgun avec `url: 'https://api.eu.mailgun.net'`                                                                                         |
| **Variables d'env**  | `MAILGUN_API_KEY`, `MAILGUN_DOMAIN=mail.attendee.fr`, (SMTP : `MG_SMTP_HOST/PORT/USER/PASSWORD`) → **EAS Secrets / .env prod**, jamais en clair |
| **Service d'envoi**  | `MailgunEmailService` injectable (voir §5)                                                                                                      |
| **Queue**            | job `email.send` + processor, `jobId` déterministe, retry + backoff                                                                             |
| **Throttle dynamique** | réglage Redis/DB du débit global + pause/reprise ; idéalement seuils par domaine destinataire                                                  |
| **Templates**        | migrer les emails (confirmation + billet PDF joint) ; vérifier taille attachment                                                                |
| **Webhooks**         | endpoint recevant `accepted`/`delivered`/`deferred`/`bounce`/`complained`/`blocked`/`failed` → table de suivi                                    |
| **Monitoring domaine** | agrégation par domaine destinataire : Gmail, Outlook/Hotmail, Yahoo, domaines pro                                                              |
| **Tests**            | staging via Mailpit ou domaine sandbox Mailgun ; test technique 3 000 en test mode ; test réel progressif sur seed list                         |
| **Bascule**          | basculer le transport une fois `mail.attendee.fr` vérifié (fin de C1)                                                                           |

## 4. Templates & pièce jointe

- 1 email transactionnel par inscription : **confirmation + billet PDF joint** (+ **lien R2 signé** de secours).
- Envoi **après** génération du billet PDF (séquencement déjà acté au diagnostic).
- Vérifier la **limite de taille des attachments** via API (SMTP OK partout).

## 5. Snippets de référence (région EU)

> Source : rapport de décision [ChoixESP-transactionnel-pour-Attendee-LFD.pdf](./ChoixESP-transactionnel-pour-Attendee-LFD.pdf).

### API HTTP (recommandé) — service NestJS

```ts
import { Injectable } from "@nestjs/common";
import Mailgun from "mailgun.js";
import FormData from "form-data";

@Injectable()
export class MailgunEmailService {
  private readonly client = new Mailgun(FormData).client({
    username: "api",
    key: process.env.MAILGUN_API_KEY!,
    url: "https://api.eu.mailgun.net", // région EU
  });

  async sendConfirmation(to: string) {
    return this.client.messages.create(process.env.MAILGUN_DOMAIN!, {
      from: `Attendee <no-reply@${process.env.MAILGUN_DOMAIN}>`,
      to: [to],
      subject: "Confirmation d'inscription",
      html: "<p>Votre inscription est confirmée.</p>",
    });
  }
}
```

### Alternative — transport Nodemailer via SMTP relay

```ts
import * as nodemailer from "nodemailer";

export const mailgunTransport = nodemailer.createTransport({
  host: process.env.MG_SMTP_HOST!, // hôte SMTP région EU (dashboard)
  port: Number(process.env.MG_SMTP_PORT ?? 587),
  secure: false,
  auth: {
    user: process.env.MG_SMTP_USER!, // souvent postmaster@mail.attendee.fr
    pass: process.env.MG_SMTP_PASSWORD!,
  },
});
```

> Mailgun recommande l'**API HTTP** plutôt que le SMTP à l'échelle → privilégier le 1er snippet.

## 6. Checklist de bascule (fin C2, après C1 vérifié)

- [ ] Domaine `mail.attendee.fr` **Verified** (dépend de C1 / accès OVH)
- [ ] Secrets Mailgun en prod (pas en clair)
- [ ] Worker email séparé de l'API, branché sur la queue
- [ ] Débit email pilotable à chaud (Redis/DB) sans restart API
- [ ] Pause/reprise email disponible pendant l'event
- [ ] Webhooks reçus et enregistrés en base
- [ ] Events Mailgun tracés : `accepted`, `delivered`, `deferred`, `bounce`, `complained`/`spam`, `blocked`/`failed`
- [ ] Retry Mailgun observé sur cas `deferred`
- [ ] Monitoring délivrabilité + file BullMQ en place
- [ ] Monitoring par domaine destinataire en place (Gmail, Outlook/Hotmail, Yahoo, domaines pro)
- [ ] Test technique 3 000 envois en `o:testmode=true`
- [ ] Test réel progressif OK (seed list + quelques centaines si audience légitime)
- [ ] Confirmation support Mailgun : 12 400 emails / 2 jours, pic submit ~20 emails/s, région EU, PDF attaché
- [ ] Bascule du transport prod → Mailgun
