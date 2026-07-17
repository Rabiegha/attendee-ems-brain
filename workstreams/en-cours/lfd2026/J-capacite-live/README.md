# Chantier J — Capacité live forte charge

> Portier Redis (anti-survente) + WebSocket statut live + pic combiné inscription/check-in.
> Cadré dans le [workstream 02 — capacité live forte charge](../../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md).

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (ligne J du tableau)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **J**)
- **Owner :** Corentin
- **Statut :** 🟡 socle partiellement livré via H/Corentin — recoupe H/I, ne pas double-compter (S30–S31)

## Réévaluation 2026-07-17

Avancement J réévalué à **~35 %**.

Ce qui existe déjà côté code, livré surtout dans le chantier H :

- Verrou pessimiste DB `FOR UPDATE` sur la ligne `sessions` pendant l'inscription publique à une session.
- Logique capacité/waitlist avec refus `409` si complet sans waitlist.
- Inscription idempotente si le même attendee se réinscrit à la même session.
- Calcul de stats live après inscription : `registered`, `capacity`, `seats_left`, `state`.
- Diffusion WebSocket back vers la room publique `public-session:<token>` via `public-session:stats`.
- Tests E2E H-T7/H-T8/H-T9/H-T9b qui couvrent inscription nominale, idempotence, complet sans waitlist et waitlist.

Ce qui reste dans J :

- Décider si le verrou DB suffit pour l'event ou s'il faut ajouter un portier Redis / compteur atomique.
- Ajouter un cache public léger pour éviter de recalculer les jauges trop souvent sous forte charge.
- Brancher le front public sur `public-session:stats` ou mettre un polling robuste si le WS public n'est pas retenu.
- Prévoir la stratégie multi-instance WebSocket : Socket.IO Redis adapter ou garantie d'un seul process WS pendant l'event.
- Tester le pic combiné inscription session + check-in + consultation jauges.
- Ajouter métriques, alertes et runbook court pour surveiller surbooking, latence inscription et erreurs WS.

Décision de suivi : le temps déjà passé reste imputé à H. J garde seulement le reste haute charge / validation / multi-instance.
