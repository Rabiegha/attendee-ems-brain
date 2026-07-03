# Rabbit Hole — Billeterie publique : app séparée + temps réel

Type: Rabbit Hole
Status: En cours (décision cadrée, arbitrages actés)
Risk level: High
Date: 2026-07-03

> **Contexte client :** un organisateur veut piloter sa **billeterie** et exposer une **page publique
> par session** (lien public par session). Pic attendu : **~3000 concurrents**. Le plafond d'écriture
> mesuré d'Attendee est **~33–37 inscriptions/s** (sérialisation DB) — voir
> [workstreams/en-cours/api-scaling-lfd2026](../../workstreams/en-cours/api-scaling-lfd2026/diagnostic-email-billet-wallet.md).

---

## 1. Pourquoi c'est tentant (et pourquoi ça peut déraper)

- « On a presque tout sur Attendee » : **event = billeterie**, **sessions = jauges**, **badge creator = créateur de billet**, capacité déjà gérée. Tentation : tout empiler dans le monolithe.
- Tentation inverse : créer une nouvelle app → sentiment de « ne pas faire avancer Attendee ».
- Tentation temps réel : mettre du **WebSocket public** pour refléter l'ouverture/fermeture des jauges en direct.
- Tentation générique : construire un **builder de site web** dès maintenant (custom domains, etc.).

Chacune de ces pentes mène à une usine à gaz ou à un incident de prod au pic.

## 2. Les pièges à ne PAS creuser

- 🚫 **WebSocket public** pour la page billeterie (3000 connexions persistantes, Redis adapter + sticky sessions obligatoires, sur une gateway **sans scale-out** aujourd'hui — [QandA/13](../../workstreams/a-faire/async-architecture/QandA/13-information-gaps.md) — partagée avec print-client/organisateur).
- 🚫 **Polling naïf** tapant Postgres (3000 clients / 5 s = ~600 req/s de lecture sur le chemin déjà saturé en écriture).
- 🚫 **Dupliquer la logique métier** dans l'app séparée (jauge, capacité, anti-surbooking).
- 🚫 **2ᵉ surface d'admin** : piloter les jauges à la fois depuis Attendee ET depuis le back-office landing.
- 🚫 **Builder de site web générique** maintenant (DNS/TLS auto = [full-custom-domains.md](../a-faire/full-custom-domains.md)).

## 3. Décision cadrée (2026-07-03)

Voir la billeterie comme **deux couches**, pas « une feature de plus » :

| Sujet | Décision |
|---|---|
| Où vit le métier | **Attendee** (event/sessions/badge = billeterie/jauges/billet) |
| Pilotage organisateur | **Attendee** — onglet Sessions renommé « Jauges/Créneaux » (**un seul endroit**) |
| Page billeterie publique | **App séparée** = vitrine qui consomme l'API publique Attendee |
| Ce qu'on ajoute à Attendee | **API billeterie publique** + **cache état jauge** + **compteur atomique Redis** |
| Temps réel public | **Polling depuis cache (TTL 10–20 s)**, jitteré — **pas de WS public** |
| WebSocket | Réservé au **dashboard organisateur authentifié** (déjà en place) |
| Anti-surbooking | À l'**écriture** (Redis DECR + réconciliation) ; affichage « **presque complet** » près de la limite |
| Périmètre client | Version pragmatique, **pas** de builder générique — garder un **seam propre** (API publique versionnée) |

### Principe directeur — découpler lecture et écriture

- État « ouvert / fermé / presque complet » servi depuis un **cache (Redis/edge/CDN, TTL court)**, jamais une requête DB par appel.
- Les écritures (inscription, toggle ouvrir/fermer) **mettent à jour / invalident** le cache.
- Le pic public tape **Redis/CDN**, pas Postgres. Postgres ne voit que les ~33 écritures/s (plafond réel).
- L'affichage peut être **eventually-consistent** à quelques secondes ; l'atomicité anti-surbooking se joue **à l'écriture**, pas à l'affichage.
- **Bénéfice résilience** : la vitrine sert une page cachée même si Attendee est sous pression (aligné [plan-continuite-activite.md](../../workstreams/en-cours/api-scaling-lfd2026/plan-continuite-activite.md)).

## 4. Pourquoi l'app séparée est le bon choix (pas un pis-aller)

1. **Profil de trafic opposé** : public/massif/cacheable vs back-office authentifié. Isoler = protéger le back-office et la gateway WS du pic public.
2. **« Site web » sera réutilisé** (landing pages — [public-landing-page-namespaces.md](../../backlog/a-faire/public-landing-page-namespaces.md)). Un canal de diffusion publique dédié est son bon foyer.
3. **Anti-usine-à-gaz** : le monolithe n'absorbe pas un frontend public + ses soucis d'edge/cache. Attendee ne reçoit qu'une **couche API publique mince**.

> On **fait bien avancer Attendee** : on lui ajoute une API billeterie publique + le compteur de jauge (primitive de scaling nécessaire de toute façon). L'app séparée est un **canal**, pas une redite.

## 5. Quand revisiter / élargir

- Si le besoin dépasse « lien public par session » (parcours event agrégé multi-sessions → phase 3 du workstream sessions).
- Si le temps réel « à la seconde » devient contractuel (billets en quasi-rupture) → alors seulement évaluer SSE/WS **dédié et scale-out**, jamais sur la gateway actuelle.
- Si plusieurs clients demandent le « site web » → généraliser à partir du seam propre (pas avant).

## 6. Related docs

- [../../workstreams/a-faire/sessions-inscriptions-lfd2026/README.md](../../workstreams/a-faire/sessions-inscriptions-lfd2026/README.md) — primitives jauge/session (ouvrir/fermer, capacité, token public, stats live).
- [../../workstreams/en-cours/api-scaling-lfd2026/diagnostic-email-billet-wallet.md](../../workstreams/en-cours/api-scaling-lfd2026/diagnostic-email-billet-wallet.md) — plafond écriture + compteur Redis DECR.
- [../../workstreams/en-cours/api-scaling-lfd2026/plan-continuite-activite.md](../../workstreams/en-cours/api-scaling-lfd2026/plan-continuite-activite.md) — jauges & continuité au pic.
- [../../bugs/a-faire/cors-origin-security-review.md](../../bugs/a-faire/cors-origin-security-review.md) — CORS/Origin WS (origin `*` aujourd'hui).
- [../a-faire/full-custom-domains.md](../a-faire/full-custom-domains.md) — piège builder générique / DNS-TLS.
- [../a-faire/separate-admin-frontend.md](../a-faire/separate-admin-frontend.md) — piège « 2ᵉ frontend » (distinct : ici l'app publique est justifiée).

## 7. Rule for Copilot

- Si une proposition contient « **WebSocket** pour la page billeterie **publique** », « le front public **poll la DB** », « **dupliquer** la logique de jauge dans l'app billeterie », « piloter les jauges **aussi** depuis le back-office landing » → **stop**, pointer vers ce document.
- Autorisé : API publique Attendee (lecture cache-first) + compteur Redis + app vitrine mince qui consomme l'API.
- Interdit : coupler la page publique au chemin d'écriture ; ouvrir un canal WS public sur la gateway `/events` actuelle ; construire un builder de site web générique pour ce client.
