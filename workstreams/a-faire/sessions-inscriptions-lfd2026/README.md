# Workstream — Inscriptions & refonte des sessions (LFD 2026)

> **Objet :** permettre l'**inscription publique au niveau d'une session** (pas seulement de
> l'event), avec un **lien public par session**, une gestion **capacité + liste d'attente**, et une
> **refonte front des sessions** (inspirée de la section *Placement*) donnant à l'organisateur
> **visibilité + contrôle** en temps réel.
> **Rattachement :** événement La Fabrique de la Diplomatie 2026 (MEAE) —
> nouveau **Chantier** du [00-plan-action.md](../../en-cours/lfd2026/00-plan-action.md).
> **Date :** 1ᵉʳ juillet 2026 · **Statut :** 🟡 cadrage validé, prêt à découper.

---

## 0. Flux cible

```
Inscription sur une SESSION  →  crée/complète l'inscription sur l'EVENT  →  la registration est
rattachée à cette session (inscrit). Le check-in physique dans la session reste un scan sur site.
```

**Décision sémantique actée :** l'inscription via un lien session **inscrit** la personne à la
session (`SessionRegistration`, ex-`RegistrationSessionChoice`). Elle **ne crée pas** de présence
physique — le `SessionScan admitted` reste le check-in réel, fait sur site le jour-J.

---

## 1. Pilier de conception — le mode pilote tout, la capacité est optionnelle

> **Acté (01/07) :** une session **sans inscription/choix n'a aucun intérêt à avoir un lien public**.
> Le lien public vit donc **sous** le mode « avec inscription ». La capacité reste un **sous-axe
> optionnel** à l'intérieur de ce mode.

| Mode (`registration_enabled`) | Inscription / choix | Lien public | Capacité | Stats affichées |
| --- | --- | --- | --- | --- |
| **`false` — walk-in** (défaut, existant) | ❌ aucune | ❌ **aucun** (inutile) | — | **check-ins uniquement** |
| **`true` — avec inscription** (LFD) | ✅ oui | ✅ dispo (`public_registration_enabled` + `public_token` + `public_registration_open`) | **optionnelle** : illimitée (`capacity = null`) **ou** limitée (`capacity` + `capacity_enforced` + `waitlist_enabled`) | **inscrits + check-ins + progression + répartition par type** |

> Le rappel du cadrage reste vrai : par défaut, quiconque est inscrit à l'event peut entrer dans une
> session walk-in (on ne compte que les check-ins). Dès qu'une session passe en `registration_enabled`,
> elle gère ses propres inscrits, expose (si voulu) son lien public, et peut être **illimitée ou
> plafonnée**.

> ⚠️ **Impact UI :** cartes/stats/écrans s'**adaptent au mode**. Walk-in → tout l'inscriptionnel est
> masqué, on n'affiche que les check-ins.

---

## §B-bis. Pourquoi token par session (choix long terme)

> **Principe : primitive vs. vue.** Le **token par session** est une *primitive* (brique de base) ;
> le **lien event unique avec sélecteur** est une *vue* (agrégateur) qu'on construit **par-dessus**.
> Avec la primitive, on peut toujours ajouter ensuite une page « lien event » qui liste les sessions.
> L'inverse impose de **rétro-ajouter les tokens** (migration + refactor). ⇒ on construit la
> primitive d'abord ; l'agrégateur (**phase 3**) vient sans rework.
>
> Bénéfices du token par session : ciblage marketing par session (campagne/QR/partenaire),
> ouverture/fermeture + capacité **indépendantes** par session, attribution UTM propre, gestion
> naturelle des **modes mixtes** (le token n'existe que sur les sessions sur inscription). L'unique
> contrepartie — « plus de liens à gérer » — est neutralisée par l'UI façon *Placement* (bouton
> copier/activer/stopper sur chaque carte). Le **lien event unique** reste souhaitable pour le
> parcours « je parcours le programme et je choisis » → livré en **phase 3** au-dessus des mêmes
> endpoints par session.

---

## 2. Décisions de cadrage (actées)

| # | Décision | Choix retenu |
| --- | --- | --- |
| A | Sens de « inscrit à la session » | **Inscription** (`SessionRegistration`), pas de scan auto. Le scan présence reste sur site. |
| B | Modèle du lien public par session | **Token dédié par session** (`Session.public_token`) + route `/public/sessions/:token/register`. *Reco retenue — voir §B-bis.* |
| C | Multi-session pour un même email | **Oui** — **une seule registration event** qui **accumule** les sessions. ⇒ **assouplir l'anti-doublon** actuel. |
| D | Capacité à l'inscription | **Event + session** contrôlées à l'inscription ; **waitlist** si session pleine ; capacité **modifiable** et **blocage désactivable**. |
| E | Contrôle organisateur (cahier des charges) | **Stopper** les inscriptions, **bloquer** une inscription manuellement, **augmenter** la capacité, **promouvoir** depuis la waitlist. |
| F | Front | **Refonte** inspirée de *Placement* : cartes + barre de progression → **page détail** par session (liste + config + stats live). |

---

## 3. État actuel (constats vérifiés)

- **Un seul token public, au niveau event** — `EventSetting.public_token`
  ([schema.prisma](../../../../attendee-ems-back/prisma/schema.prisma#L507)). Aucun token/route session.
- **Le lien registration↔session existe déjà** (`RegistrationSessionChoice`, PK `registration_id +
  session_id`, `priority`) — [schema.prisma](../../../../attendee-ems-back/prisma/schema.prisma#L795).
- **🐛 Bug à corriger :** le DTO public accepte `session_choice_ids`
  ([public.service.ts](../../../../attendee-ems-back/src/modules/public/public.service.ts#L359)) **mais
  ne le persiste jamais** (seul `table_choice_ids` est écrit dans la transaction). → valeur
  silencieusement jetée.
- **Anti-doublon bloquant** : toute 2ᵉ inscription du même email est rejetée
  ([public.service.ts](../../../../attendee-ems-back/src/modules/public/public.service.ts#L456)) →
  incompatible avec le multi-session (décision C).
- **3 notions de présence distinctes** : `RegistrationSessionChoice` (inscrit) · `SessionScan`
  (check-in session, IN/OUT) · `Registration.checked_in_at` (check-in event).
- **Front sessions** tout inline dans
  [EventSessionsTab.tsx](../../../../attendee-ems-front/src/pages/EventDetails/EventSessionsTab.tsx) :
  `name, location, capacity, start_at/end_at, allowedAttendeeTypes, restrict_by_choice`. Pas de page
  détail, pas de lien public session, pas de stats live, pas de liste d'inscrits.
- **Patterns réutilisables (Placement)** : cartes + barre de capacité color-codée, panneau config,
  polling react-query + refresh manuel, endpoint stats `/tables/satisfaction`, gardes de permission
  `events.update/read` — [EventPlacementTab.tsx](../../../../attendee-ems-front/src/pages/EventDetails/EventPlacementTab.tsx),
  [event-tables.service.ts](../../../../attendee-ems-back/src/modules/event-tables/event-tables.service.ts).

---

## 4. Modèle de données (Prisma) — évolutions

### 4.1 `Session` — nouveaux champs

| Champ | Type | Rôle |
| --- | --- | --- |
| `registration_enabled` | `Boolean @default(false)` | **Mode** de la session (§1) |
| `public_token` | `String? @unique` | Lien public dédié (généré quand inscription publique activée) |
| `public_registration_open` | `Boolean @default(false)` | **Stopper/ouvrir** les inscriptions sans supprimer le token |
| `waitlist_enabled` | `Boolean @default(false)` | Comportement quand la capacité est atteinte |
| `capacity_enforced` | `Boolean @default(true)` | Pouvoir **désactiver le blocage** capacité |
| `speakers` | `Json?` | Contenu riche (intervenants) — *phase 3* |
| `agenda` | `Json?` | Programme détaillé — *phase 3* |
| `confirmation_template_id` | `String? @db.Uuid` | Email de confirmation dédié — *phase 3* |
| `form_config` | `Json?` | Formulaire/champs spécifiques session — *phase 3* |

> `location`, `capacity`, `description`, `allowedAttendeeTypes`, `restrict_by_choice` existent déjà.

### 4.2 `RegistrationSessionChoice` → registre d'inscription session

Faire **évoluer le modèle existant** (moins destructif qu'un nouveau) — il devient l'**inscription
session** :

| Champ | Type | Rôle |
| --- | --- | --- |
| `status` | `SessionRegistrationStatus` | `confirmed` \| `waitlisted` \| `blocked` \| `cancelled` |
| `source` | `String?` | `public_form` \| `manual` \| `import` |
| `created_at` / `updated_at` | `DateTime` | Audit + progression temporelle des inscriptions |

PK inchangée (`registration_id, session_id`). Check-in session = **toujours** `SessionScan` (inchangé).

> **Migration :** backfill `status = confirmed`, `source = 'import'` pour l'existant. Aucune donnée
> perdue.

---

## 5. Backend — routes

### 5.1 Public (cœur LFD)
- `GET  /public/sessions/:sessionToken` — métadonnées session pour le formulaire public.
- `POST /public/sessions/:sessionToken/register` — inscription :
  1. résoudre session + event via `Session.public_token` ;
  2. vérifier `public_registration_open` + event ouvert ;
  3. **anti-doublon assoupli** : upsert attendee → registration event **réutilisée si existante**
     (on **ajoute** la session), sinon créée ;
  4. capacité **event + session** ; si session pleine → `waitlisted` (si activé) sinon `409` ;
  5. upsert `SessionRegistration` (`confirmed`/`waitlisted`, `source='public_form'`) ;
  6. emails via le pipeline BullMQ existant (template session si défini).
- Réutiliser les optimisations de charge déjà en place (transaction courte, cf. levier L9).

### 5.2 Admin sessions (nouveaux)
- `GET   /events/:eventId/sessions/:id/registrations` — liste inscrits (paginée/filtrée) + statut check-in.
- `GET   /events/:eventId/sessions/:id/stats` — stats live (voir §7).
- `POST  /events/:eventId/sessions/:id/registrations` — inscrire manuellement une registration.
- `PATCH /events/:eventId/sessions/:id/registrations/:registrationId` — statut (block / unblock / promote waitlist).
- `DELETE /events/:eventId/sessions/:id/registrations/:registrationId` — retirer.
- `POST/PATCH /events/:eventId/sessions/:id/public-token` — générer / activer / stopper le lien.

---

## 6. Front — refonte (inspiration *Placement*)

- **Grille de cartes session** avec **barre de progression** *mode-aware* :
  - mode *ouverte* → barre = **check-ins** ;
  - mode *inscription* → barre = **inscrits / capacité** + badge waitlist.
- **Clic carte → page détail session** avec onglets : **Inscriptions** · **Check-ins** ·
  **Paramètres** · **Stats**.
- **Panneau config** : mode inscription, lien public (copier / activer / **stopper**),
  capacité + waitlist, types autorisés, description riche, email dédié (*ph.3*), form dédié (*ph.3*).
- **Contrôles organisateur** (visibilité + contrôle) : stop inscriptions, changer capacité,
  bloquer une inscription, promouvoir depuis la waitlist.
- **Live** : polling react-query + refresh (pattern Placement) en phase 1–2 ; **websocket** en phase 3.
- **Public** : page `/register/session/:sessionToken` affichant les specs de la session.

---

## 7. Stats par session (mode-aware, temps réel)

| Métrique | Mode ouverte | Mode inscription |
| --- | --- | --- |
| **Check-ins** (uniques, via `SessionScan admitted`) | ✅ métrique principale | ✅ |
| **Inscrits** (`confirmed`) | ❌ masqué | ✅ |
| **Waitlist** (`waitlisted`) | ❌ | ✅ |
| **Répartition par type** (présents / inscrits) | présents seulement | inscrits **et** présents |
| **Progression** (inscriptions dans le temps) | ❌ | ✅ (courbe via `created_at`) |

> Calcul on-demand (comme `calculateEventStats` / `/tables/satisfaction`), pas de cache. Websocket
> possible en phase 3 pour le live sans refresh.

---

## 8. Phasage proposé

| Phase | Contenu | Pourquoi |
| --- | --- | --- |
| **0 — Fondations data** | Migration Prisma (§4) + backfill + **fix bug `session_choice_ids`** | Débloque tout le reste |
| **1 — Inscription publique par session (back)** | Token session + routes publiques + anti-doublon assoupli + capacité/waitlist | **Cœur LFD** — priorité |
| **2 — Refonte front sessions** | Cartes + page détail + config + liste inscrits + stats live (mode-aware) | Visibilité + contrôle organisateur |
| **3 — Confort** | Form dédié session, emails dédiés, contenu riche, websocket live | Non bloquant pour l'event |

> **À trancher pour le plan LFD :** phases 0–2 sont candidates au périmètre du mois ; la phase 3 est
> reportable post-event. À insérer comme **nouveau Chantier** dans
> [00-plan-action.md](../../en-cours/lfd2026/00-plan-action.md).

> 🔴 **Forte charge (3000 simultanés + anti-survente + live) :** l'approche « polling + capacité en
> base » ci-dessus **ne tient pas** sous pic. Design dédié (Redis portier atomique + WebSocket),
> **à démarrer APRÈS chantier A (les leviers)** → [02-capacite-live-forte-charge.md](02-capacite-live-forte-charge.md).

---

## 9. Points de vigilance

- **Anti-doublon (décision C)** : bien distinguer « déjà inscrit à **cet event** » (autorisé, on
  ajoute la session) de « déjà inscrit à **cette session** » (idempotent). Ne pas rejouer le rejet actuel.
- **Course capacité/waitlist** : le `COUNT` capacité est best-effort hors transaction aujourd'hui —
  sous pic LFD, décider si la promotion waitlist doit être sérialisée (advisory lock, comme le scan session).
  → sous forte charge, remplacé par le **portier atomique Redis** ([02](02-capacite-live-forte-charge.md)).
- **Ne pas fausser les compteurs** : l'inscription **ne crée pas** de `SessionScan` (décision A).
- **Perf** : réutiliser les leviers du workstream `infra-scaling-pca` (tx courte, worker BullMQ,
  cache) — ne pas réintroduire une transaction interactive longue dans le register session.
- **Permissions** : `events.update` (config/contrôle) et `events.read` (stats/listes), comme Placement.
