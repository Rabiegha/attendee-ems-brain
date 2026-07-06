# Workstream — Infra Scaling & Plan de Continuité (LFD 2026, contrat MEAE)

> **Statut :** À FAIRE — diagnostic terminé, optimisations partielles livrées ; reste = leviers perf (chantier I) + PCA/HA.
> **Priorité :** 🔴 Haute (contrat client, échéance 4–5 sept 2026)
> **Dernière session :** 2026-06-25
> **Repo principal :** `attendee-ems-back`
> **Contenu :** perf/scaling + **plan de continuité** ([plan-continuite-activite.md](./plan-continuite-activite.md)) + audit staging ([journal/](journal/2026-06-26-audit-branche-staging.md)).
> **Workstream événement lié** (email/billet/PDF/sessions) : [lfd2026](../../en-cours/lfd2026/README.md).

---

## Où est l'info détaillée

Tout le détail technique est dans **[infra/](../../../infra/)** (ems-brain) :

- **[lfd-2026-session-handoff-2026-06-25.md](../../../infra/lfd-2026-session-handoff-2026-06-25.md)** ⭐ — **commencer par là** (handoff complet,
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

## ⚡ Pourquoi ça plafonne à ~30/s (diagnostic prouvé — LE plus gros levier)

Prouvé par nos propres mesures (25/06) : ce n'est **ni le CPU** (2→4 cœurs = même débit),
**ni le pool DB** (1–4 connexions actives/100), **ni l'email** (worker à 8 %). C'est la
**transaction Prisma d'inscription** qui tient une connexion pendant un **`COUNT` de
capacité** + upsert + create → **sérialisation**. C'est un plafond **algorithmique**, donc
**améliorable sans changer d'infra** : compteur de capacité **dénormalisé** (UPDATE atomique
ou Redis DECR) + **transaction réduite au `create`**. **Gain plausible ×3–×10.**
Cadré comme **chantier I** dans [00-plan-action.md](../../en-cours/lfd2026/00-plan-action.md) (§3-I).
Tant que ce levier n'est pas fait, le plafond reste — quelle que soit l'infra (OVH, GCP, VM plus grosse).

---

## Reste à faire (priorisé)

1. **⚡ Chantier I — compteur capacité dénormalisé + transaction courte** = **LE plus gros levier** pour > 37 reg/s (cf. section ci-dessus + plan d'action §3-I).
2. Mesure propre k6 (machine externe, pas le VPS).
3. Déployer Voie A (worker séparé) en prod.
4. Spike test 3000 VUs (capacité max).
5. Cluster (Voie B) **seulement après** workstream clustering.
6. Corriger doc executive-summary (« badge PDF » → QR).

---

## ⚠️ Règle absolue

**Voie B (cluster N instances) = JAMAIS en prod** tant que clustering (redis-adapter + sticky +
présence Redis) non livré → casse l'impression temps réel silencieusement.
