# C2.1 — Orchestration billet PDF + email session + suivi minimal

> **Objet :** faire le pont entre C2, B et BIL : orchestrer la generation du billet PDF
> apres inscription session, afficher un suivi web minimal au participant, puis envoyer l'email
> billet via l'infra Mailgun/BullMQ existante.

- **Parent :** [C — Migration Mailgun](../README.md)
- **Depend de :** [B — Email -> billet PDF](../../B-email-billet-pdf/README.md), dont
  [B2 — fallback Puppeteer resilient](../../B-email-billet-pdf/B2-fallback-puppeteer-resilient.md)
- **Depend aussi de :** [BIL — Plateforme billetterie](../../BIL-billetterie/README.md) pour les decisions produit billet/QR/parcours
- **Suivi transversal :** [03-suivi-chantiers.md](../../03-suivi-chantiers.md) ligne **C2.1**
- **Inclus V1 :** page confirmation persistante, reload safe, polling leger, statut billet/email,
  bouton telechargement billet quand pret, messages email en attente/envoye.
- **Hors perimetre volontaire :** websocket de progression billet/email, correction email
  self-service, resend avance, verification email obligatoire.

## Fichiers

| Fichier | Sujet |
| --- | --- |
| [00-suivi.md](./00-suivi.md) | Statut, decisions, points ouverts et prochaine action |
| [01-etat-des-lieux.md](./01-etat-des-lieux.md) | Ce que C2 permet deja, ce qui manque pour les billets session |
| [02-plan.md](./02-plan.md) | Plan C2.1 : orchestration billet PDF + email, suivi minimal, workers, configuration, risques |

## Decision de cadrage

C2.1 n'est pas un levier de capacite type L9/L9.1. C'est une extension applicative de C2 :

```txt
C2  = transport Mailgun + queue email + webhooks + throttle
B   = generation PDF / Gotenberg
BIL = decisions produit billet / QR / parcours billetterie
C2.1 = orchestration billet PDF + email session + suivi minimal entre ces briques
```

Le PDF ne doit jamais etre genere dans la requete HTTP d'inscription. L'inscription session
reste le chemin critique. Le billet et l'email sont traites en asynchrone.

UX incluse :

```txt
Inscription confirmee
-> billet en preparation
-> billet pret + telechargement
-> email en attente / envoye
```

Decision worker :

```txt
worker-pdf separe : requis pour C2.1 et partage avec B2
worker-email separe : requis cible, a faire avant gros volume si possible
api-web : garde les inscriptions synchrones et transactionnelles
```
