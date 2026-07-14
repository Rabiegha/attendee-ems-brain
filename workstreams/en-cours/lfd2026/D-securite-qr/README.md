# Chantier D — Sécurité QR (HMAC + durcissement routes)

> Signature HMAC des QR + durcissement de la route publique de check-in.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-D)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **D**)
- **Owner :** Corentin — **Statut (14/07) :** 🟢 back **en prod de facto** (merges staging→main 0-CI du 13/07) ·
  mobile mergé sur `main` (PR #9) mais **⏸️ release mobile en stand-by** (décision 14/07 : release unique à la fin
  du chantier mobile à venir — voir [04-etat-branches §Mobile](../04-etat-branches.md))
- **⚠️ Vigilance (vérifié dans le code le 14/07) :** le back en prod est **déjà strict** (UUID brut refusé au scan,
  by design `e96e7b8`). La 1.1.7 reste compatible : elle relaie la chaîne scannée au back, et les QR des emails
  sont re-signés côté serveur à l'affichage (même les vieux emails). Offline 1.1.7 : nom absent du feedback
  (match local par UUID impossible sur un token signé) mais le replay passe. **Seul vrai risque : les supports
  figés d'avant le 13/07** (badges déjà imprimés, PDF stockés, images QR cachées client mail — ancien cache 1 an)
  → rejetés par le back → **confirmer qu'aucun ne servira pour l'event** · poser `QR_HMAC_SECRET` dédié en prod
  (fallback dérivé de `JWT_ACCESS_SECRET` actif)
- **Reste :** E2E scan QR signé bout-en-bout · release mobile (fin du chantier mobile) · secret dédié prod
