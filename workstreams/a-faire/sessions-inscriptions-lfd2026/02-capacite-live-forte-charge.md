# 02 — Capacité live sous forte charge (Redis portier + WebSocket)

> **Rattachement :** partie du workstream [Inscriptions & refonte des sessions (LFD 2026)](README.md).
> **Objet :** tenir **3000 inscriptions quasi simultanées** sur des sessions à **capacité limitée**,
> avec **statut live** (pleine/fermée) affiché aux participants — **sans survente** et **sans saturer
> Postgres**.
> **Date :** 2026-07-08 · **Statut :** 🟡 cadré, **à démarrer APRÈS chantier A (les leviers)**.

---

## 0. Pourquoi cette partie existe (vs. le §6/§7 du README)

Le README de ce workstream prévoit des **stats on-demand + polling react-query** (phase 1-2) et
« websocket en phase 3 ». **C'est suffisant pour la charge normale**, mais **pas** pour le scénario
LFD **3000 simultanés + capacité serrée** :

- **polling naïf** → 1000 req/s sur Postgres = mort ;
- **contrôle capacité en base** (ligne chaude verrouillée par 3000 requêtes) → sérialisation + **survente** ;
- **stats on-demand** → recalcul à chaque poll, trop coûteux sous pic.

Cette partie **remplace** l'approche « polling + capacité en base » **pour les sessions à forte
demande**. Les sessions normales gardent l'approche simple du README.

> **Séquencement :** cette partie vient **après chantier A** (leviers L9/L7/L8/L3 + baseline k6).
> On mesure d'abord ; on ne construit ce dispositif que si la mesure confirme le besoin.
> Détails conceptuels : [learning — pic d'inscription temps réel](../../../learnings/2026-07-08-pic-inscription-temps-reel.md).

---

## 1. Principe : séparer LECTURE et ÉCRITURE

| Chemin | Trafic | Destination | Plafond |
|---|---|---|---|
| **Lecture** (statut/poll) | ~1000 req/s | **Redis / cache / WebSocket** | énorme |
| **Écriture** (inscription) | rafale de 3000 | **file BullMQ → Postgres** | ~33/s |

**Règle d'or :** le chemin lecture ne touche **jamais** Postgres.

---

## 2a. Portier atomique Redis (anti-survente) 🔴 cœur

- clé par session : `session:{id}:places` initialisée à la capacité.
- à l'inscription : `DECR` atomique.
  - **≥ 0** → accepté → **enfile** la persistance (BullMQ).
  - **< 0** → **rejeté « complet »** instantané (+ `INCR` compensatoire non requis : on ne persiste rien).
- **waitlist** : si `< 0` et `waitlist_enabled`, pousser dans une liste Redis `session:{id}:waitlist`
  (statut `waitlisted`, cf. modèle §4.2 du README).
- **source de vérité = Redis pendant le pic**, réconcilié en base à la persistance.

> **Invariant :** aucune inscription n'est confirmée sans un `DECR ≥ 0`. La base ne fait que
> **matérialiser** des décisions déjà prises atomiquement par Redis.

### Points de vigilance 2a
- **Init/reconciliation** : au démarrage, seeder `places` depuis la base (capacité − inscrits confirmés).
- **Modif capacité par l'orga** (décision E du README) : mettre à jour la clé Redis **et** la base.
- **Annulation / désinscription** : `INCR` pour rendre la place + promotion waitlist.
- **`capacity_enforced = false`** : bypass du portier (illimité) → ne pas décrémenter.
- **Idempotence** : re-soumission du même email ne doit pas re-décrémenter (clé de dédup par
  `registration_id + session_id`).

---

## 2b. Transport temps réel (affichage live) 🟢

- **WebSocket (socket.io)** via un `@WebSocketGateway` NestJS ; **rooms par session**
  (`session:{id}`) → broadcast ciblé, pas aux 3000.
- **émission** au **changement significatif** : ouvert → presque plein → complet → fermé
  (throttle max 1x/1-2s), **pas** un broadcast par inscription.
- l'affichage a **droit à 1-2s de retard** : la correction est garantie par le portier 2a au submit.
- **replis** : SSE (push unidirectionnel plus simple) ou **polling caché** (cache HTTP/CDN 1-2s TTL).

### Pièges config (à ne pas oublier)
- **`ulimit -n`** (`LimitNOFILE`) : défaut 1024 → monter au-delà du nb de connexions.
- **nginx** : `Upgrade`/`Connection` + `proxy_read_timeout` longs.
- **mono-instance obligatoire** tant que pas de **redis-adapter** (multi-instance = post-event,
  cf. [scaling horizontal stateless](../../../learnings/2026-07-08-scaling-horizontal-stateless.md)).

---

## 3. Ordre de construction (la vérité avant le tuyau)

| # | Brique | Dépend de |
|---|---|---|
| 1 | **Portier Redis 2a** + init/reconciliation depuis la base | — |
| 2 | **File BullMQ** persistance inscription (si pas déjà via L7/L8) | levier L7/L8 |
| 3 | **Émission d'events** au changement de statut | 1 |
| 4 | **Gateway WebSocket + rooms** (back) | 3 |
| 5 | **Front** : abonnement + MAJ UI | 4 |
| 6 | **Config infra** : `ulimit`, nginx upgrade/timeout | 4 |
| 7 | **Test de charge** : N clients WS + rafale d'inscriptions | tout |

---

## 4. Critères de test (gate avant prod)

- [ ] **Pas de survente** : lancer > capacité en parallèle → exactement `capacity` confirmés, reste
      `complet`/`waitlisted`.
- [ ] **Postgres protégé** : sous rafale 3000, la DB reste à ~débit drainé (pas de pic de verrous).
- [ ] **Lecture ne touche pas la DB** : les polls/statuts sont servis par Redis (vérifié en métriques).
- [ ] **Propagation live** : un changement de statut se voit côté front en < 2s.
- [ ] **Reconnexion** : redémarrage instance → clients se reconnectent sans tempête (backoff).
- [ ] **Réconciliation** : compteur Redis == base après le pic (aucune place fantôme).

---

## 5. Hors périmètre event (post-event)

- **Multi-instance** (redis-adapter socket.io + sticky nginx) → chantier
  [api-scaling-clustering](../../en-cours/api-scaling-clustering/README.md).
- Pour l'event : **mono-instance verticale** suffit.
