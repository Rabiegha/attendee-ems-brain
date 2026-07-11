# Chantier C — Migration Mailgun (délivrabilité de masse)

> **Décision : Mailgun** (région EU, IP partagée, API HTTP + queue BullMQ) pour la délivrabilité
> des ~12 400 envois. Découpé en **C1 (prépa Mailgun + warm-up)** et **C2 (migration applicative)**.
> 🔴 **Warm-up domaine à lancer tôt** (~2 sem min, 3-4 idéal) — délai calendaire incompressible.
> ⛔ **Bloqueur C1 : accès OVH (DNS)** requis pour SPF/DKIM/DMARC + vérif du domaine `mail.attendee.fr`.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-C1 / §3-C2)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (lignes **C1** et **C2**)

## Fichiers du dossier

| Fichier                                                                                          | Sujet                                                                     |
| ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| [esp-choix-mailgun.md](./esp-choix-mailgun.md)                                                   | Décision — comparatif ESP + **pourquoi Mailgun**                          |
| [MAILGUN_WARMUP_PLAN.md](./MAILGUN_WARMUP_PLAN.md)                                               | **C1** — prépa Mailgun + warm-up domaine + validation (phases 1/2/3)      |
| [mailgun-migration-technique.md](./mailgun-migration-technique.md)                               | **C2** — intégration NestJS (API EU, queue, templates, webhooks, bascule) |
| [ChoixESP-transactionnel-pour-Attendee-LFD.pdf](./ChoixESP-transactionnel-pour-Attendee-LFD.pdf) | Rapport source de la décision                                             |
