# Workstream — Scaling API & charge LFD 2026 (contrat MEAE)

> **Statut :** EN COURS — diagnostic terminé, optimisations partielles livrées
> **Priorité :** 🔴 Haute (contrat client, échéance 4–5 sept 2026)
> **Dernière session :** 2026-06-25
> **Repo principal :** `attendee-ems-back`

---

## Où est l'info détaillée

Tout le détail technique est dans **`attendee-ems-back/docs/infra/`** :

- **`lfd-2026-session-handoff-2026-06-25.md`** ⭐ — **commencer par là** (handoff complet,
  déroulé, chiffres, reste à faire, règles critiques, infos opé).
- `lfd-2026-load-test-results.md` — résultats techniques mesurés.
- `lfd-2026-reproduction.md` — refaire les optimisations + tests soi-même.
- `lfd-2026-client-report.md` — rapport client MEAE.
- `lfd-2026-load-test-plan.md`, `lfd-2026-capacity-planning.md` — plan & dimensionnement.

Workstreams techniques liés (back) : `docs/workstreams/async-badge-email/`,
`docs/workstreams/api-scaling-clustering/`.

---

## Résumé en 5 lignes

- Besoin : tenir **3 000 inscriptions simultanées** au pic (CDC § 2.3 / § 6.1).
- Mesuré : **~33 inscriptions/s** mono-instance (plancher) → 3 000 absorbées en **~90 s**. OK.
- **Plafond = sérialisation écriture DB** (ni CPU, ni pool, ni nb de process — prouvé).
- Optimisations livrées : pool DB, email→BullMQ, transaction allégée, worker séparable (`PROCESS_ROLE`).
- Cluster testé (+robustesse) mais **interdit en prod** sans le workstream clustering.

---

## Reste à faire (priorisé)

1. **Optimiser la transaction d'écriture** (sortir le COUNT capacité) = LE levier pour > 37 reg/s.
2. Mesure propre k6 (machine externe, pas le VPS).
3. Déployer Voie A (worker séparé) en prod.
4. Spike test 3000 VUs (capacité max).
5. Cluster (Voie B) **seulement après** workstream clustering.
6. Corriger doc executive-summary (« badge PDF » → QR).

---

## ⚠️ Règle absolue

**Voie B (cluster N instances) = JAMAIS en prod** tant que clustering (redis-adapter + sticky +
présence Redis) non livré → casse l'impression temps réel silencieusement.
