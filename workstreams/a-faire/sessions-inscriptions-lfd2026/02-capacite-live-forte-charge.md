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

## 2c. Démarrage à froid (première visite = 3000 en même temps) 🔴

À la première visite, chaque user charge **2 choses** : le **bundle front** (statique) + un **read
initial** (état sessions). Deux pièges :

- **Bundle** → **CDN / statique**, jamais l'API (sinon 3000 téléchargements concurrencent l'API).
- **Read sur cache FROID → cache stampede** : 3000 requêtes sur cache vide, non coordonnées, passent
  toutes jusqu'à Postgres.

**Parades :**
1. **Pré-chauffer le cache** (`session:*` + payloads dérivés) **avant l'ouverture** des inscriptions.
2. **Single-flight** : sur cache froid, une seule requête reconstruit, les autres attendent (verrou court).

---

## 2d. Cohérence pull + push (le point subtil)

**Cache (pull)** et **WebSocket (push)** = **deux mécanismes séparés**, **même vérité Redis** :

- **Pull = snapshot** (à l'arrivée) · **Push = stream** (les changements). Client : pull **puis** subscribe.
- **Deux usages de Redis** : *store-vérité* (`places`, `open` → écriture directe, pas d'invalidation) vs
  *cache dérivé* (listes/stats → write-through + TTL court).
- **Règles anti-divergence :** (1) un seul écrivain (l'API, dans Redis) ; (2) **écrire Redis PUIS émettre** ;
  (3) event WS = **état absolu** (« places = 12 »), pas delta → tolérant aux pertes/désordre.
- **Mono vs multi :** mono-instance = **emit direct** (le WS n'écoute pas Redis) ; multi-instance =
  **Redis pub/sub** (redis-adapter) → post-event uniquement.

> Détail conceptuel complet : [learning — pic d'inscription temps réel](../../../learnings/2026-07-08-pic-inscription-temps-reel.md).

---

## 2e. Pic COMBINÉ : inscriptions + check-in session 🔴 angle mort des mesures

Les mesures (~25/s CPU, ~33/s DB) étaient pour l'**inscription isolée**. Le jour J, **plusieurs flux
d'écriture** tapent **en même temps** — le vrai plafond, c'est leur **somme** :

| Flux | Quoi | Latence tolérée | Chemin |
|---|---|---|---|
| **Inscription session** | s'inscrire à une session | ⏳ quelques sec OK | **file BullMQ** |
| **Check-in session** (`SessionScan`) | scan d'entrée **dans une session** | ⚡ immédiat | **HTTP synchrone** |
| **Check-in entrée** (`checked_in_at`) | scan d'entrée de l'**event** | ⚡ immédiat | **HTTP synchrone** |

> ⚠️ **Les 3 notions de présence sont distinctes** (déjà acté §3 du README). Le check-in session ≠
> check-in entrée.

### Conséquences
1. **La mesure isolée sous-estime la charge réelle** → tester le **scénario combiné** en k6 (trafic
   mixte : inscriptions + scans session + scans entrée simultanés). Trou dans le plan actuel →
   [load-test-plan](../../../infra/lfd-2026-load-test-plan.md).
2. **Le check-in ne peut PAS être mis en file** (personne à la porte → « admis/refusé » immédiat).
   → il faut **isoler + prioriser** le chemin check-in face à la rafale d'inscriptions.

### ⚠️ Piège de vocabulaire — le check-in ne se « worker-ise » PAS

Un **worker** traite des jobs **en différé** (asynchrone). Le **check-in est synchrone** : la personne
est **à la porte** et attend « admis/refusé » en < 1s. → **on ne met JAMAIS le check-in dans une file.**

« Prioriser le check-in » ≠ lui donner un worker. Ça veut dire **protéger son chemin HTTP synchrone**
contre la rafale d'inscriptions. Ce qu'on met dans le worker, c'est l'**inscription** (qui, elle,
tolère du différé) — ce qui **libère** la DB **pour** le check-in.

| Flux | Nature | Va dans une file ? |
|---|---|---|
| Inscription | asynchrone (« c'est en cours ») | ✅ oui → worker BullMQ |
| Check-in | **synchrone** (personne à la porte) | ❌ **jamais** → HTTP direct |

> **Worker = 1 process, plusieurs *types* de jobs — pas 1 worker par tâche.** Un seul worker consomme
> toute la file (register, email, PDF…). On n'ajoute pas un worker par fonctionnalité ; on ajoute des
> **types de jobs**. C'est une archi **durable**, pas une rustine. Ce qui est temporaire, c'est le
> **mono-instance** (limite verticale), pas le worker.

### Isolation du check-in — faisable JOUR J, SANS clustering
> **Worker séparé ≠ clustering.** Un worker BullMQ (L8) est une **séparation de rôles** mono-machine
> (l'API sert le HTTP, le worker vide la file). Il ne tient **ni socket ni imprimante** → **aucun**
> problème d'état. Le clustering (L13, interdit avant tests verts) concerne **plusieurs copies du rôle
> stateful** — pas ça.

Deux leviers retenus, **tous mono-machine et durables** :
1. **La file isole déjà (gratuit)** : les inscriptions dans BullMQ n'occupent pas les threads HTTP des
   check-ins ; le worker draine à ~33/s **en arrière-plan**.
2. **Budget de connexions DB réservé** au check-in (pools pgBouncer distincts) → une rafale d'inscriptions
   ne peut pas **assécher** les connexions du check-in. **= config, pas de code, et ça se garde à long
   terme** (chaque instance garde ses pools cloisonnés, même en clustering plus tard).
3. **(Optionnel, non retenu pour l'instant)** process API dédié check-in — endpoints **sans état**, donc
   OK sur une machine, toujours **pas** du clustering. Laissé de côté tant que (1)+(2) suffisent.

---

## 2f. Waitlist & confirmation asynchrone

### Waitlist — back = vérité, front = affichage
- **Back** (propriétaire) : `DECR < 0` + `waitlist_enabled` → `status=waitlisted` (+ liste Redis) ;
  gère **position** et **promotion** (annulation → `INCR` → promeut la tête de file → `confirmed` → notifie).
- **Front** : affiche « en liste, position N » ; écoute le WS → « une place s'est libérée, confirmé ».

### Fin d'inscription — confirmation synchrone, travail async
- **Immédiat (sync)** : `DECR ≥ 0` **garantit la place** → réponse HTTP « ✅ inscrit » **tout de suite**
  (pas besoin d'attendre l'écriture DB).
- **Différé (async)** : persistance Postgres + email + PDF badge → **file BullMQ**.
- **Pas du fire-and-forget aveugle** : BullMQ **retry** + **dead-letter** → rien perdu silencieusement.
- **Notif front de fin** : email (billet) et/ou **push WS** « billet prêt » si confort temps réel voulu.

> UX cible : **place confirmée instantanément** (portier), **billet qui arrive juste après** (file).

---

## 2g. Levier candidat **L9.1** — compteur de présence session O(1)

> ⚠️ **Conditionné à la mesure.** À ne déclencher que si un test montre que le `COUNT` gêne
> (table `session_scans` grosse + pic combiné). Le check-in étant gated par l'humain, il se peut
> qu'il n'y ait **rien à faire**.

**Problème actuel** ([sessions.service.ts](../../../../attendee-ems-back/src/modules/sessions/sessions.service.ts)) :
à chaque scan **IN** sur une session à capacité, le code fait **`2× COUNT(*)`** sur `session_scans`
(admitted IN / admitted OUT) → **O(n)** : le coût **grandit** avec le nombre de scans déjà enregistrés.

**Levier** : arrêter de recompter → tenir un **compteur incrémental** (`+1` IN admis / `−1` OUT admis).
Lecture de la capacité = **lire un nombre = O(1)**, constant quelle que soit la taille de la table.

**Où stocker le compteur — deux options, les DEUX règlent le O(n) :**

| Option | Règle le O(n) ? | Quand |
|---|---|---|
| **Colonne Postgres `present_count`** (défaut) | ✅ | **Suffit ici.** Check-in gated humain (quelques scans/s) → contention négligeable. Simple, **cohérent** (même tx que l'insert du scan). |
| **Redis `session:{id}:present`** (nice-to-have) | ✅ | **Win « no effort »** : Redis sera **déjà en place pour le portier inscription (2a)** → réutiliser `INCR`/`DECR` coûte quasi rien, et donne en **bonus** : lecture **hors Postgres** + compteur **temps réel poussable en WS**. Rôle = **cache dérivé** (PG = vérité, seed + réconciliation). |

> **Redis n'est PAS obligatoire pour ce levier** — c'est juste **où** ranger le compteur. Il ne devient
> indispensable que sous **forte concurrence sur la même clé** (= le portier inscription, 3000 simultanés),
> pas pour le check-in session. **Mais** comme il tourne déjà pour l'inscription, l'y brancher est un
> **gain gratuit** si on veut l'affichage live d'occupation.

**Durable** : le O(1) reste constant à toute échelle et taille de table, et **survit au clustering**.

> ⇒ Asymétrie à retenir : le **check-in session** a ce problème O(n) ; le **check-in event principal**
> ([`checkIn()`](../../../../attendee-ems-back/src/modules/registrations/registrations.service.ts)) **non**
> (simple `UPDATE` par clé primaire sur sa propre ligne, aucun `COUNT`, aucune ligne chaude) → déjà O(1),
> **pas de levier**, juste l'isolation par pool DB réservé (§2e).

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
| 7 | **Endpoint read (snapshot)** servi par Redis + **pré-chauffe** + single-flight | 1 |
| 8 | **CDN** pour le bundle front (ou nginx statique + cache headers — suffit à 3000) | — |
| 9 | **Isolation check-in** : pool DB réservé (pgBouncer) + priorité sur la rafale inscriptions | levier L2 |
| 9b | **Compteur présence session** (L9.1) : colonne PG `present_count` O(1) — **si la mesure le justifie** ; Redis en bonus si l'affichage live est voulu | mesure |
| 10 | **Test de charge COMBINÉ** : WS + inscriptions (file) + check-ins session/entrée (sync) simultanés | tout |

---

## 4. Critères de test (gate avant prod)

- [ ] **Pas de survente** : lancer > capacité en parallèle → exactement `capacity` confirmés, reste
      `complet`/`waitlisted`.
- [ ] **Postgres protégé** : sous rafale 3000, la DB reste à ~débit drainé (pas de pic de verrous).
- [ ] **Lecture ne touche pas la DB** : les polls/statuts sont servis par Redis (vérifié en métriques).
- [ ] **Propagation live** : un changement de statut se voit côté front en < 2s.
- [ ] **Reconnexion** : redémarrage instance → clients se reconnectent sans tempête (backoff).
- [ ] **Réconciliation** : compteur Redis == base après le pic (aucune place fantôme).
- [ ] **Démarrage à froid** : 3000 arrivées simultanées sur cache pré-chauffé → 0 (ou ~1) requête
      Postgres pour le read initial (single-flight vérifié).
- [ ] **Cohérence pull/push** : après un changement, pull et push renvoient la **même** valeur
      (état absolu, écriture Redis avant émission).
- [ ] **Pic combiné** : sous inscriptions (rafale) **+** check-ins session/entrée simultanés, les
      check-ins gardent une latence < 1s (le chemin sync n'est pas affamé par la file).
- [ ] **Waitlist** : session pleine → `waitlisted` ; annulation → promotion auto de la tête de file + notif.
- [ ] **Confirmation** : place confirmée en sync (portier) ; échec de persistance → retry BullMQ (pas de perte).

---

## 5. Hors périmètre event (post-event)

- **Multi-instance** (redis-adapter socket.io + sticky nginx) → chantier
  [api-scaling-clustering](../../en-cours/api-scaling-clustering/README.md).
- Pour l'event : **mono-instance verticale** suffit.
