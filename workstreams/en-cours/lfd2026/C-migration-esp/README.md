# Chantier C — Migration Mailgun (délivrabilité de masse)

> **Décision : Mailgun** (région EU, IP partagée, API HTTP + queue BullMQ) pour la délivrabilité
> des ~12 400 envois. Découpé en **C1 (prépa Mailgun + DNS)**, **C2 (migration applicative)**,
> **C2.1 (orchestration billet PDF + email session)** et **C3 (warm-up / délivrabilité opérationnelle)**.
> 🔴 **Warm-up domaine à lancer tôt** (~2 sem min, 3-4 idéal) — délai calendaire incompressible.
> ✅ **C1 terminé 15/07 :** plan Mailgun acheté, domaine EU `mail.attendee.fr` vérifié,
> DNS OVH publiés, env staging/prod posés, tests d'envoi OK.
> ⚙️ **Décision 14/07 : C2 doit intégrer un rate-limit email pilotable à chaud** (Redis/DB,
> sans restart API), les webhooks Mailgun (`accepted`, `delivered`, `deferred`, `bounce`,
> `complained`/`spam`, `blocked`/`failed`) et un monitoring par domaine destinataire
> (Gmail, Outlook/Hotmail, Yahoo, domaines pro).

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-C1 / §3-C2)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (lignes **C1**, **C2**, **C2.1**, **C3**)

## Fichiers du dossier

| Fichier                                                                                          | Sujet                                                                     |
| ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| [esp-choix-mailgun.md](./esp-choix-mailgun.md)                                                   | Décision — comparatif ESP + **pourquoi Mailgun**                          |
| [MAILGUN_WARMUP_PLAN.md](./MAILGUN_WARMUP_PLAN.md)                                               | **C1** — prépa Mailgun + warm-up domaine + validation (phases 1/2/3)      |
| [mailgun-migration-technique.md](./mailgun-migration-technique.md)                               | **C2** — intégration NestJS (API EU, queue, templates, webhooks, bascule) |
| [C2-1-email-billet-session/](./C2-1-email-billet-session/)                                       | **C2.1** — orchestrer billet PDF + email session en réutilisant C2, B et BIL |
| [c3-warmup-delivrabilite.md](./c3-warmup-delivrabilite.md)                                       | **C3** — suivi opérationnel warm-up, webhooks, métriques, runbook         |
| [warm-up-strategie.md](./warm-up-strategie.md)                                                   | Stratégie d'envoi warm-up avec base newsletter Channel Scope              |
| [ChoixESP-transactionnel-pour-Attendee-LFD.pdf](./ChoixESP-transactionnel-pour-Attendee-LFD.pdf) | Rapport source de la décision                                             |
