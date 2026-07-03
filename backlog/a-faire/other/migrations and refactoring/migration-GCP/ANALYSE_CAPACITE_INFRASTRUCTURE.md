# Analyse de Capacité Infrastructure — Attendee EMS

> **Date :** 17 avril 2026  
> **Cible :** Google Cloud Platform — VM unique + Cloud SQL  
> **Application :** Attendee EMS (SaaS multi-tenant de gestion d'événements)

---

## Table des matières

1. [Récapitulatif de l'infrastructure cible](#1-récapitulatif-de-linfrastructure-cible)
2. [Compréhension de l'application et des flux](#2-compréhension-de-lapplication-et-des-flux)
3. [Profil de consommation des ressources](#3-profil-de-consommation-des-ressources)
4. [Analyse par scénario (Bas / Normal / Extrême)](#4-analyse-par-scénario)
5. [Verdict sur le sizing VM](#5-verdict-sur-le-sizing-vm)
6. [Premiers goulots d'étranglement](#6-premiers-goulots-détranglement)
7. [Endpoints et traitements à risque de saturation](#7-endpoints-et-traitements-à-risque-de-saturation)
8. [Signaux de scaling — Quand agir et quoi faire](#8-signaux-de-scaling--quand-agir-et-quoi-faire)
9. [Prérequis techniques pour que l'infra tienne](#9-prérequis-techniques-pour-que-linfra-tienne)
10. [Estimation des coûts et évolution](#10-estimation-des-coûts-et-évolution)

---

## 1. Récapitulatif de l'infrastructure cible

| Composant | Spécification |
|-----------|--------------|
| **Cloud Provider** | Google Cloud Platform |
| **Compute** | 1 VM Compute Engine (e2-standard-2 ou e2-standard-4) |
| **VM Sizing** | Option A : 2 vCPU / 4 Go RAM — Option B : 4 vCPU / 8 Go RAM |
| **Stockage** | SSD persistant 50–100 Go |
| **Base de données** | Cloud SQL PostgreSQL (managé, séparé de la VM) |
| **Reverse Proxy** | Nginx (conteneurisé sur la VM) |
| **Déploiement** | Docker Compose (API NestJS + Nginx) |
| **Load Balancer** | ❌ Pas au départ |
| **CDN** | ❌ Pas au départ |
| **Auto-scaling** | ❌ Pas au départ |
| **HA** | ❌ Pas au départ |
| **CI/CD** | Vers la VM (GitHub Actions → SSH deploy) |
| **Sécurité** | IAM, Firewall, Secrets Manager, TLS, SSH contrôlé |
| **Monitoring** | Cloud Monitoring + Alerts |

### Ce qui tourne sur la VM

| Service | Rôle | Mémoire estimée au repos |
|---------|------|-------------------------|
| **Docker Engine** | Orchestration conteneurs | ~100–200 Mo |
| **Nginx** | Reverse proxy + frontend statique | ~30–50 Mo |
| **NestJS API** | Backend applicatif + serving HTML badges | ~300–500 Mo |
| **Chromium (Puppeteer)** | Export PDF uniquement (PAS impression) | ~150–250 Mo (si instance persistante, 0 si lazy-load) |
| **OS (Linux)** | Système de base | ~200–300 Mo |
| **Total au repos** | | **~800 Mo – 1,3 Go** |

### Ce qui NE tourne PAS sur la VM

- ✅ PostgreSQL → Cloud SQL (séparé)
- ✅ EMS Print Client → Machines locales des utilisateurs (Electron)
- ✅ App Mobile → Appareils des utilisateurs
- ✅ Frontend → Fichiers statiques servis par Nginx (pas de SSR)

---

## 2. Compréhension de l'application et des flux

### Architecture multi-tenant

```
Organisation (Tenant/Client)
  ├── Users (Admin, Manager, Coordinator, Hostess)
  ├── Events
  │   ├── Registrations (inscriptions)
  │   │   └── Attendees (participants)
  │   ├── Sessions / Sous-événements
  │   ├── Tables (placement physique)
  │   ├── Badge Templates (modèles de badges)
  │   └── Email Templates
  └── Configuration (types, tags, secteurs)
```

### Cycle de vie d'un événement

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PHASE PRÉPARATION (jours/semaines)               │
│  Usage: FAIBLE                                                       │
│  • Création événement, configuration                                 │
│  • Import listes Excel (bulk)                                        │
│  • Configuration badges, emails, tables                              │
│  • Consultation / analyse des attendees                              │
│  Actions: CRUD classique, quelques imports, consultation             │
│  Charge: < 5 req/s                                                   │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     PHASE CHECK-IN (30 min – 1h)                     │
│  Usage: PIC MAXIMAL                                                  │
│  • Scan QR code (mobile/web)                                         │
│  • Check-in API call                                                 │
│  • Attribution table automatique                                     │
│  • Broadcast WebSocket (temps réel)                                  │
│  • Webhook n8n (fire-and-forget)                                     │
│  • Création print job → WebSocket → Print Client                    │
│  • Possiblement: génération badge PDF à la volée                     │
│  Charge: 0.5 – 3+ req/s de check-in + WebSocket broadcasts          │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     PHASE POST-EVENT                                 │
│  Usage: FAIBLE                                                       │
│  • Consultation stats, export données                                │
│  • Analyses, scoring                                                 │
│  Actions: Lectures DB, exports                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Détail d'un check-in (action critique)

```
Scan QR Code
    │
    ▼
POST /registrations/{id}/check-in
    │
    ├─ 1. Lecture registration + event + attendee (1 query DB ~5ms)
    ├─ 2. Validations métier (~1ms CPU)
    ├─ 3. UPDATE registration: checked_in_at, checked_in_by (~3ms DB)
    ├─ 4. Auto-assign table (1 query + 1 update ~5ms DB)
    ├─ 5. WebSocket broadcast org room (~0.1ms)
    ├─ 6. [ASYNC] POST webhook n8n (~50ms réseau, non-bloquant)
    ├─ 7. [ASYNC] Recalcul scoring/affinité (~10-50ms, non-bloquant)
    └─ 8. [ASYNC] Création PrintJob + notif WS (~2ms DB + broadcast)
    │
    ▼
Réponse HTTP: ~15-30ms (hors badge)
```

### Détail d'une impression de badge (flow réel)

L'impression **n'utilise PAS Puppeteer côté serveur**. Le serveur génère du HTML rendu, et c'est le **Print Client Electron** (sur la machine locale de l'utilisateur) qui fait le rendu + impression.

```
Mobile/Web → POST /print-queue/add {registrationId, printerName}
    │
    ├─ 1. Création PrintJob en DB (~3ms)
    ├─ 2. Broadcast WebSocket 'print-job:created' (~0.1ms)
    │
    ▼
Print Client Electron (machine locale) reçoit le job via WebSocket
    │
    ├─ 3. GET /badge-generation/{badgeId}/metadata (~5ms serveur)
    ├─ 4. GET /badge-generation/{badgeId}/html (~15-30ms serveur)
    │      ├─ Résolution template (cascade de priorités) (~5ms DB)
    │      ├─ Remplacement variables {{firstName}}, {{tableName}}... (~2ms CPU)
    │      ├─ Génération QR code base64 (~10ms CPU)
    │      └─ Retourne HTML complet avec styles inline
    │
    ├─ 5. [LOCAL] Écriture HTML dans fichier temp
    ├─ 6. [LOCAL] Chargement dans BrowserWindow Electron (Chromium local)
    ├─ 7. [LOCAL] webContents.print() → imprimante physique (SANS PDF)
    └─ 8. PATCH /print-queue/{id}/complete (~3ms serveur)
    │
    ▼
Charge serveur par badge: ~20-40ms CPU, ~0 RAM supplémentaire
Charge lourde (rendu Chromium): sur la machine du Print Client, PAS le serveur
```

### Détail d'une génération PDF de badge (opération lourde, OPTIONNELLE)

La génération PDF via Puppeteer **existe mais est séparée** du flow d'impression. Elle sert pour l'export/download de badges, PAS pour l'impression directe.

```
POST /badges/generate/{registrationId}
    │
    ├─ 1. Résolution template (~5ms DB)
    ├─ 2. Remplacement variables (~2ms CPU)
    ├─ 3. Génération QR code base64 (~10ms CPU)
    ├─ 4. Ouverture page Puppeteer (~100-200ms, +20-50 Mo RAM)
    ├─ 5. Injection HTML + rendu CSS (~200-500ms)
    ├─ 6. Export PDF page.pdf() (~300-500ms)
    ├─ 7. Upload Cloudflare R2 (~100-300ms réseau)
    └─ 8. Fermeture page Puppeteer (libération mémoire)
    │
    ▼
Durée totale: 700ms – 2s par badge
Mémoire pic: +20-50 Mo par badge en cours de rendu
⚠️ Ceci n'est PAS déclenché pendant le check-in / impression
```

---

## 3. Profil de consommation des ressources

### CPU

| Opération | Intensité CPU | Fréquence |
|-----------|--------------|-----------|
| Check-in API | Faible (~5ms) | Haute en pic |
| Génération HTML badge (sans Puppeteer) | **Faible** (~20-40ms) | Haute en pic (impression) |
| Génération PDF badge (Puppeteer, optionnel) | **Élevée** (~500ms-2s) | Rare (export/download) |
| Import Excel bulk | Moyenne (parsing + N inserts) | Ponctuelle |
| Recalcul scoring/affinité | Moyenne | Async, post check-in |
| CRUD standard (events, attendees) | Faible | Régulière |
| WebSocket broadcast | Négligeable | Haute en pic |
| QR code generation | Faible (~10ms) | Liée aux badges |

### Mémoire

| Composant | Repos | Sous charge |
|-----------|-------|-------------|
| NestJS API (Node.js) | 300–500 Mo | 600 Mo – 1,5 Go |
| Chromium (Puppeteer, instance persistante) | 150–250 Mo | 250–500 Mo |
| Par page Puppeteer active (export PDF uniquement) | — | +20–50 Mo |
| Connexions WebSocket (par connexion) | ~10 Ko | ~10 Ko |
| 100 connexions WS simultanées | ~1 Mo | ~1 Mo |
| Nginx | 30–50 Mo | 50–100 Mo |
| Import Excel (buffer en mémoire) | — | +10–50 Mo par fichier |

### Réseau (depuis la VM)

| Flux | Direction | Volume |
|------|-----------|--------|
| Réponses API → clients | Egress | Faible (JSON, ~1-10 Ko/réponse) |
| Badge PDF → Print Client | Egress | Moyen (~100-500 Ko/badge) |
| Upload badge → Cloudflare R2 | Egress | Moyen (~100-500 Ko/badge) |
| WebSocket frames | Bidirectionnel | Faible (~0.5-2 Ko/event) |
| Webhook n8n | Egress | Faible (~1 Ko/check-in) |
| API → Cloud SQL | Interne GCP | Faible-Moyen (requêtes SQL) |

### Connexions base de données

| Paramètre | Valeur actuelle |
|-----------|----------------|
| Pool Prisma par défaut | ~10 connexions |
| Par check-in | 2-4 queries |
| Check-in rate max (10/s) | ~20-40 queries/s |
| Import 1000 lignes | ~2000-3000 queries (séquentiel) |

---

## 4. Analyse par scénario

### 📊 Scénario BAS — Lancement / Premiers clients

| Paramètre | Valeur |
|-----------|--------|
| **Tenants actifs** | 3–5 |
| **Users total** | 10–20 |
| **Events/semaine** | 2–5 |
| **Taille events** | 100–300 inscrits |
| **Events simultanés (pic)** | 1 |
| **Check-in simultanés** | 1 seul event à la fois |
| **Rate check-in pic** | 300 en 30min = **10/min = 0.17/s** |
| **Connexions WebSocket pic** | 5–10 (quelques écrans + mobile) |
| **Badges imprimés** | 100–300 (HTML servi, rendu local sur Print Client) |
| **Imports Excel** | 1–3 fichiers/semaine, <500 lignes |
| **Print Clients connectés** | 1–2 |

#### Verdict scénario BAS

| Ressource | 2 vCPU / 4 Go | 4 vCPU / 8 Go |
|-----------|---------------|---------------|
| CPU | ✅ 3–10% moyen, 15–25% pic | ✅ <5% moyen |
| RAM | ✅ ~1,2 Go utilisé (~30%) | ✅ ~1,2 Go utilisé (~15%) |
| DB connexions | ✅ <5 actives | ✅ <5 actives |
| Stockage | ✅ <10 Go | ✅ <10 Go |
| Réseau | ✅ <1 Go/mois egress | ✅ <1 Go/mois egress |

> **🟢 Les deux configurations supportent largement ce scénario.** Le 2 vCPU / 4 Go est suffisant.

---

### 📊 Scénario NORMAL — Production établie

| Paramètre | Valeur |
|-----------|--------|
| **Tenants actifs** | 10–25 |
| **Users total** | 50–100 |
| **Events/semaine** | 10–20 |
| **Taille events** | 500–2000 inscrits |
| **Events simultanés (pic)** | 2–3 (jours de pointe) |
| **Check-in simultanés** | 2 events en même temps |
| **Rate check-in pic** | 2×1000 en 30min = **67/min = 1.1/s** |
| **Connexions WebSocket pic** | 20–40 |
| **Badges imprimés** | 1000–2000/jour (HTML servi, rendu local) |
| **Pré-génération PDF** | Non nécessaire — l'impression utilise du HTML direct |
| **Imports Excel** | 5–10/semaine, 500–2000 lignes |
| **Print Clients connectés** | 2–5 |

#### Charge de travail pendant le pic (30 min de check-in)

```
Check-in rate: 1.1/s
  → 1.1 × (2 DB reads + 1 DB write + 1 table assign) = ~5 queries/s
  → 1.1 × WebSocket broadcast = 1.1 events/s
  → 1.1 × async webhook = 1.1 HTTP POST/s
  → 1.1 × print job creation = 1.1 inserts/s + WS notif

Impression badges (HTML direct, PAS Puppeteer):
  → 1.1 × GET /badge-generation/{id}/html = 1.1 requêtes/s
  → Chaque requête: ~20-40ms CPU serveur (template + QR code)
  → Le rendu lourd se fait sur le Print Client Electron (machine locale)
  → Pas d'impact mémoire significatif côté serveur
  → ✅ Le serveur peut facilement servir 10+ badges HTML/s
```

#### Verdict scénario NORMAL

| Ressource | 2 vCPU / 4 Go | 4 vCPU / 8 Go |
|-----------|---------------|---------------|
| CPU (check-in + impression HTML) | ✅ 15–30% pic | ✅ 10–20% pic |
| CPU (si export PDF batch en parallèle) | ⚠️ 50–70% pic | ✅ 30–45% pic |
| RAM (check-in + impression HTML) | ✅ ~1,5 Go (~38%) | ✅ ~1,5 Go (~19%) |
| RAM (si export PDF batch en parallèle) | ⚠️ 2–3 Go (~55-75%) | ✅ 2–3 Go (~28-38%) |
| DB connexions | ✅ 5–10 actives | ✅ 5–10 actives |
| Stockage | ✅ 15–30 Go | ✅ 15–30 Go |
| Réseau egress | ✅ 5–15 Go/mois | ✅ 5–15 Go/mois |

> **� Le 2 vCPU / 4 Go tient bien — l'impression utilise du HTML léger, pas Puppeteer.**  
> **🟢 Le 4 vCPU / 8 Go est très confortable pour ce scénario.**  
> **⚠️ Risque uniquement si un export PDF batch (Puppeteer) est lancé pendant un pic de check-in.**

---

### 📊 Scénario EXTRÊME — Croissance soutenue

| Paramètre | Valeur |
|-----------|--------|
| **Tenants actifs** | 30–50+ |
| **Users total** | 150–300 |
| **Events/semaine** | 30+ |
| **Taille events** | 2000–5000+ inscrits |
| **Events simultanés (pic)** | 3–5 |
| **Check-in simultanés** | 3 events en même temps |
| **Rate check-in pic** | 3×5000 en 1h = **250/min = 4.2/s** |
| **Connexions WebSocket pic** | 50–100+ |
| **Badges imprimés** | 5000–15000/jour (HTML servi, rendu local) |
| **Imports Excel** | 15+/semaine, 1000–5000 lignes |
| **Print Clients connectés** | 5–10 |

#### Charge de travail pendant le pic extrême

```
Check-in rate: 4.2/s
  → ~17 queries/s sur Cloud SQL
  → 4.2 WebSocket broadcasts/s
  → 4.2 webhooks/s (async HTTP)
  → 4.2 print jobs/s

Impression badges (HTML direct):
  → 4.2 × GET /badge-generation/{id}/html = 4.2 req/s HTML
  → ~20-40ms CPU serveur par requête
  → Charge serveur: ~15-20% CPU (requêtes légères)
  → Le goulot est le Print Client local, PAS le serveur
  → ⚠️ Avec 5+ Print Clients, la charge réseau augmente

Import 5000 lignes Excel:
  → ~10 000+ queries séquentielles
  → Durée: 30s–2min
  → Bloque une partie du pool de connexions DB
```

#### Verdict scénario EXTRÊME

| Ressource | 2 vCPU / 4 Go | 4 vCPU / 8 Go |
|-----------|---------------|---------------|
| CPU (check-in + impression HTML) | ✅ 25–40% pic | ✅ 15–25% pic |
| CPU (si export PDF batch en parallèle) | ❌ 80–100% | ⚠️ 50–70% |
| RAM (check-in + impression HTML) | ✅ ~1,8 Go (~45%) | ✅ ~1,8 Go (~23%) |
| RAM (si export PDF batch en parallèle) | ⚠️ 3–4 Go (~85%) | ✅ 3–4 Go (~45%) |
| DB connexions | ⚠️ 10–20 actives (pool limit) | ⚠️ 10–20 actives (pool limit) |
| Stockage | ✅ 30–50 Go | ✅ 30–50 Go |
| Réseau egress | ✅ 20–50 Go/mois | ✅ 20–50 Go/mois |

> **🟡 Le 2 vCPU / 4 Go peut tenir le check-in + impression HTML, mais sera serré avec les imports + DB load.**  
> **🟢 Le 4 vCPU / 8 Go tient bien ce scénario pour le check-in + impression.**  
> **⚠️ Le goulot sera la base de données (Cloud SQL) et les imports bulk, PAS Puppeteer.** Les exports PDF batch doivent être planifiés hors pic.

---

### 📊 Tableau comparatif synthétique

| Métrique | Scénario BAS | Scénario NORMAL | Scénario EXTRÊME |
|----------|:------------:|:---------------:|:----------------:|
| Tenants | 3–5 | 10–25 | 30–50+ |
| Users | 10–20 | 50–100 | 150–300 |
| Events/semaine | 2–5 | 10–20 | 30+ |
| Inscrits/event | 100–300 | 500–2000 | 2000–5000+ |
| Check-in/s (pic) | 0.17 | 1.1 | 4.2 |
| DB queries/s (pic) | <5 | 5–15 | 17–50 |
| WS connexions (pic) | 5–10 | 20–40 | 50–100+ |
| Badges imprimés/jour | 100–300 | 1000–2000 | 5000–15000 |
| **2 vCPU / 4 Go** | 🟢 OK | 🟢 OK | 🟡 Limite (DB) |
| **4 vCPU / 8 Go** | 🟢 OK++ | 🟢 OK++ | 🟢 OK |

---

## 5. Verdict sur le sizing VM

### Recommandation

| Configuration | Recommandé pour | Commentaire |
|---------------|----------------|-------------|
| **2 vCPU / 4 Go** ✅ | **Lancement viable.** Jusqu'à 15-20 tenants, events de 1000-2000 personnes | L'impression utilise du HTML léger (pas Puppeteer). Il reste ~2,5 Go pour l'API. Suffisant si Puppeteer n'est pas lancé pendant les check-ins. |
| **4 vCPU / 8 Go** ✅✅ | **Production confortable.** Jusqu'à 40 tenants, events de 3000-5000 personnes | Headroom pour les pics. Possibilité de lancer des exports PDF sans bloquer les check-ins. |
| 8 vCPU / 16 Go | Si croissance vers 50+ tenants ou events 5000+ | Avant de passer à cette taille, envisager scaling horizontal. |

### 🏆 Recommandation finale : **Le 2 vCPU / 4 Go (e2-standard-2) est viable pour le lancement**

**Justification :**

1. **L'impression utilise du HTML direct** — Le serveur ne fait que servir du HTML avec variables remplacées (~20-40ms/requête). Pas de Puppeteer pour l'impression. Le rendu lourd se fait sur le Print Client Electron (machine locale).
2. **Puppeteer n'est utilisé que pour l'export PDF** (download/archive), pas pour le flow critique de check-in. Il peut être lancé en période creuse.
3. **La charge pic serveur est avant tout I/O bound** (requêtes DB, WebSocket), pas CPU bound. Le 2 vCPU suffit.
4. Sur 4 Go, il reste ~2,5 Go pour l'API + OS (Puppeteer ~250 Mo au repos si instance persistante). C'est suffisant pour le flow d'impression HTML.
5. **Économie de ~40$/mois** au démarrage.

**Quand passer à 4 vCPU / 8 Go :**
- Si vous avez régulièrement 2+ events simultanés de 1000+ personnes
- Si vous avez besoin de lancer des exports PDF pendant les check-ins
- Si le CPU moyen dépasse 50% pendant les pics
- Si vous dépassez 20 tenants actifs

> **Si le budget le permet** : le 4 vCPU / 8 Go donne un excellent headroom et 12+ mois de croissance tranquille.

---

## 6. Premiers goulots d'étranglement

Classés par ordre de probabilité d'apparition :

### 🥇 #1 — Pool de connexions PostgreSQL / Cloud SQL

**Pourquoi c'est le premier (et non plus Puppeteer) :**
- L'impression utilise du HTML direct servi par le backend (~20-40ms, léger)
- Le rendu lourd se fait sur le Print Client Electron (machine locale), PAS le serveur
- → Le vrai goulot est la **base de données**, pas le CPU/RAM serveur
- Pool Prisma par défaut : ~10 connexions
- Un import bulk + check-in en parallèle → starvation du pool
- Cloud SQL petit tier : 100 connexions max

**Symptômes :**
- Erreurs `PrismaClientKnownRequestError: Connection pool timeout`
- Temps de réponse API > 2s sur des requêtes simples
- Timeouts intermittents pendant les pics de check-in

**Seuil critique :** > 10 requêtes DB concurrentes soutenues

### 🥈 #2 — Rate limiting Nginx (30 req/min !)

**Pourquoi :**
- La configuration actuelle limite à **30 requêtes/min** par IP sur l'API
- Le check-in mobile d'un même device = 1 req/check-in
- L'interface web d'administration fait potentiellement 10+ req/page
- Un dashboard ouvert avec polling peut consommer 5-10 req/min
- Pendant un check-in rapide, une hôtesse peut scanner 20 badges en 2 min

**Symptômes :**
- Erreurs 429 Too Many Requests côté client
- Check-ins qui échouent de manière intermittente
- Impression de lenteur aléatoire

**⚠️ Cette limite est TROP basse pour la production. Voir section 9.**

### 🥉 #3 — Génération PDF Puppeteer (si export batch pendant un pic)

**Pourquoi :**
- L'export PDF via Puppeteer **n'est PAS utilisé pour l'impression** (c'est du HTML direct)
- Mais si quelqu'un lance un export/download de badges PDF pendant un check-in → CPU saturé
- Chromium instance persistante : ~150-250 Mo RAM au repos
- ~1-2s par badge PDF, séquentiel

**Symptômes :**
- CPU 100% si export PDF + check-in en parallèle
- Toutes les réponses API ralentissent (event loop bloqué)
- Puppeteer OOM kill possible sur 4 Go RAM

**Seuil critique :** Ne JAMAIS lancer un export PDF batch pendant un check-in

### #4 — Event loop Node.js (CPU single-thread)

**Pourquoi :**
- NestJS tourne sur un seul thread principal
- Les opérations CPU-bound (parsing Excel, QR code generation, template rendering) bloquent l'event loop
- Même les check-ins simples sont impactés si Puppeteer tourne

**Symptômes :**
- Event Loop Lag > 100ms (mesurable via `perf_hooks`)
- Toutes les réponses API ralentissent simultanément
- WebSocket latence visible

**Seuil critique :** Event loop lag > 50ms de manière soutenue

### #5 — WebSocket connexions simultanées

**Seuil pas critique avant :**  
- ~500 connexions simultanées sur Socket.IO → pas un problème pour Node.js
- Devient un souci au-delà de 1000+ connexions (mémoire + file descriptors)
- **Pas un goulot pour vos cas d'usage actuels**

---

## 7. Endpoints et traitements à risque de saturation

### 🔴 Risque ÉLEVÉ

| Endpoint / Traitement | Raison | Impact |
|----------------------|--------|--------|
| `POST /badges/generate/{id}` (export PDF) | Puppeteer CPU+RAM intensif, 1-2s/badge | Bloque le thread si lancé pendant un check-in |
| `POST /registrations/import` | Parsing Excel + N inserts séquentiels | Monopolise le pool DB, bloque event loop pendant parsing |
| `POST /attendees/recalculate` | Recalcul mass de scores (timeout Nginx 180s) | CPU intensif, longue exécution |
| `GET /badge-generation/{id}/html` | Léger (~20-40ms) mais haute fréquence pendant check-in | Charge DB cumulative si 4+ check-ins/s |
| `GET /badges/{id}/pdf` (download) | Refetch R2 + pipe | I/O réseau, mais rare pendant check-in |

### 🟡 Risque MODÉRÉ

| Endpoint / Traitement | Raison | Impact |
|----------------------|--------|--------|
| `POST /registrations/{id}/check-in` | Rapide unitairement (~20ms) mais haute fréquence | Volume × fréquence = charge DB |
| `GET /registrations?event={id}` | Liste paginée avec JOINs et tri | Lent si >5000 registrations + tri sur champ JSON |
| `POST /registrations` (création unitaire) | Si auto-approve + auto-badge → chaîne d'opérations | Cascade de traitements |
| WebSocket broadcast (org room) | Si 50+ listeners → sérialisation par Socket.IO | CPU marginal mais répété |

### 🟢 Risque FAIBLE

| Endpoint / Traitement | Raison |
|----------------------|--------|
| CRUD events, attendees, tags | Simple, requêtes légères |
| Auth (login, refresh) | Rate limité à 5/min, peu fréquent |
| Configuration (templates, types) | Rare, admin uniquement |
| Health check | Trivial |

---

## 8. Signaux de scaling — Quand agir et quoi faire

### 📈 Augmentation verticale de la VM

| Signal | Seuil | Action |
|--------|-------|--------|
| CPU moyen > 60% sur 15 min | Récurrent pendant les events | Passer à la taille supérieure |
| RAM utilisée > 75% | Récurrent | Augmenter la RAM |
| Swap utilisé > 0 | Toujours problématique | Augmenter la RAM immédiatement |
| Puppeteer OOM kills | 1+ occurrence | Augmenter la RAM et limiter la concurrence |
| API response time P95 > 2s | Hors badges | VM sous-dimensionnée |

**Paliers de scaling vertical :**
```
e2-standard-2 (2 vCPU / 4 Go)  →  40$/mois
        ↓ si CPU > 60% ou RAM > 75%
e2-standard-4 (4 vCPU / 8 Go)  →  80$/mois
        ↓ si toujours insuffisant
e2-standard-8 (8 vCPU / 16 Go) → 160$/mois
        ↓ à ce stade : passer à l'horizontal
```

### 🗄️ Ajout d'un cache (Redis)

| Signal | Seuil | Quoi cacher |
|--------|-------|-------------|
| Cloud SQL CPU > 50% soutenu | Récurrent | Résultats de listes fréquentes |
| Mêmes requêtes DB répétées > 100/min | Observable dans les logs | Config event, settings, templates |
| Temps de réponse API listes > 500ms | P50 | Listes attendees avec pagination |
| Pool DB connexions saturé | > 8/10 actives | Sessions utilisateur, permissions |

**Quoi cacher en priorité :**
1. **EventSettings** (lu à chaque check-in, rarement modifié)
2. **Permissions CASL** (recalculées à chaque requête)
3. **BadgeTemplates** (lu à chaque génération de badge)
4. **Listes de tables/sessions** (lues pendant le check-in pour auto-assign)

**Coût Redis :** Memorystore (GCP) ~30-50$/mois pour 1 Go, ou Redis sur la VM (gratuit mais partage les ressources).

### 🛢️ Optimisation base de données

| Signal | Seuil | Action |
|--------|-------|--------|
| Requêtes lentes > 500ms | > 10/heure | Ajouter des index |
| Cloud SQL CPU > 60% | Soutenu | Optimiser les requêtes, ajouter EXPLAIN ANALYZE |
| Connexions Cloud SQL > 80% max | Soutenu | Augmenter le tier Cloud SQL ou optimiser le pool |
| Import bulk > 2 min pour 1000 lignes | Récurrent | Passer en batch INSERT |
| `registration` table > 500K lignes | Croissance | Partition par event ou archive |

**Optimisations prioritaires :**
1. Index composites sur `(event_id, checked_in_at)` et `(org_id, email)` — probablement déjà en place via Prisma
2. Pool DB séparé (15 conn check-in + 5 conn bulk) — voir section 9
3. Batch INSERT via `createMany()` + throttle CPU adaptatif — voir section 9
4. Activer `pgbouncer` sur Cloud SQL (connection pooling)
5. Worker thread pour le parsing Excel (libérer l'event loop)

### 🔀 Passage à plusieurs VMs + Load Balancer

| Signal | Seuil | Action |
|--------|-------|--------|
| VM e2-standard-8 ne suffit plus | CPU/RAM saturés | Scaling horizontal |
| > 3 events de 2000+ personnes simultanés | Régulier | 2 VMs + LB |
| Besoin de 0 downtime déploiement | Business critique | 2+ VMs + rolling deploy |
| > 50 tenants actifs quotidiennement | Soutenu | Dissocier services |
| Puppeteer monopolise 1 CPU complet | Régulier | Séparer le worker badge |

**Architecture horizontale recommandée :**
```
                    Load Balancer (GCP)
                    ┌────┴────┐
                VM-1 (API)    VM-2 (API)
                    │              │
                    └──── Cloud SQL ────┘
                            │
                         Redis
                    (sessions partagées)
```

**Pré-requis pour le horizontal :**
- Sessions JWT → déjà stateless ✅
- WebSocket → nécessite Redis adapter pour Socket.IO (distribuer les messages entre VMs)
- PrintQueue → registry en mémoire → doit passer en Redis
- Badge generation → doit être externalisé (Cloud Run ou worker dédié)

---

## 9. Prérequis techniques pour que l'infra tienne

### 🔴 CRITIQUES (à faire AVANT la mise en production)

#### 1. Ajuster le rate limiting Nginx

**Problème actuel :** `rate=30r/m` (0.5 req/s par IP) est beaucoup trop bas pour une utilisation normale.

**Correction recommandée :**
```nginx
# API générale : 300 req/min (5/s) avec burst de 50
limit_req_zone $binary_remote_addr zone=api:10m rate=300r/m;
limit_req zone=api burst=50 nodelay;

# Login : garder restrictif
limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;
limit_req zone=login burst=5 nodelay;

# Check-in : plus permissif (mobile scanne vite)
limit_req_zone $binary_remote_addr zone=checkin:10m rate=60r/m;
limit_req zone=checkin burst=20 nodelay;

# Badge generation : limiter pour protéger le CPU
limit_req_zone $binary_remote_addr zone=badges:10m rate=10r/m;
limit_req zone=badges burst=5 nodelay;
```

#### 2. Pool DB séparé : check-ins prioritaires vs opérations bulk

**Problème :** Un import Excel de 3000 lignes (Tenant B) peut monopoliser le pool de connexions et faire timeout les check-ins (Tenant A) — en multi-tenant, on ne peut pas bloquer les imports car on ne sait pas quel tenant a un event en cours.

**Solution : Deux pools Prisma séparés**

Les check-ins utilisent un pool prioritaire (15 connexions). Les imports/exports/recalculs utilisent un pool secondaire (5 connexions). Les deux mondes ne se touchent jamais.

```typescript
// prisma.service.ts — Ajouter un 2ème client Prisma
@Injectable()
export class PrismaService {
  // Pool principal : check-in, API temps réel, WebSocket
  readonly main: PrismaClient;

  // Pool secondaire : imports, exports PDF, recalculs scoring
  readonly bulk: PrismaClient;

  constructor() {
    this.main = new PrismaClient({
      datasources: { db: { url: process.env.DATABASE_URL + '&connection_limit=15' }}
    });
    this.bulk = new PrismaClient({
      datasources: { db: { url: process.env.DATABASE_URL + '&connection_limit=5' }}
    });
  }

  async onModuleDestroy() {
    await this.main.$disconnect();
    await this.bulk.$disconnect();
  }
}
```

**Ce qui se passe en situation réelle :**
```
Tenant A : Event 5000 personnes, check-in à fond
Tenant B : Admin importe 3000 lignes Excel

Pool principal (15 conn) → check-ins de Tenant A passent sans problème
Pool bulk (5 conn)       → import de Tenant B avance à son rythme
                           → 3000 lignes en ~1min au lieu de 30s
                           → mais les check-ins ne sont JAMAIS impactés
```

Même si 3 tenants importent en même temps, ils partagent les 5 connexions bulk. Ça ralentit les imports mais les check-ins restent intacts. Pas besoin de détecter si un event est en cours — la séparation est **structurelle**.

#### 3. Throttle CPU adaptatif : backpressure automatique

**Problème :** Node.js est single-threaded. Si un import/export consomme trop de CPU, l'event loop freeze et les check-ins timeout — même avec des pools DB séparés.

**Solution : Les opérations bulk se ralentissent automatiquement en fonction de la charge CPU.**

```
CPU < 30%  → imports/exports tournent à vitesse normale
CPU 30-50% → pause 200ms entre chaque batch
CPU 50-70% → pause 500ms entre chaque batch
CPU > 70%  → PAUSE totale, attend que ça redescende
Check-ins  → TOUJOURS prioritaires, AUCUN throttle
```

```typescript
// services/cpu-monitor.service.ts
import * as os from 'os';
import { Injectable } from '@nestjs/common';

@Injectable()
export class CpuMonitorService {
  private cpuUsage = 0;

  constructor() {
    // Échantillonne le CPU toutes les 2 secondes
    setInterval(() => this.sample(), 2000);
  }

  private sample() {
    const cpus = os.cpus();
    const total = cpus.reduce((acc, cpu) => {
      const t = Object.values(cpu.times).reduce((a, b) => a + b, 0);
      const idle = cpu.times.idle;
      return { total: acc.total + t, idle: acc.idle + idle };
    }, { total: 0, idle: 0 });
    this.cpuUsage = 1 - total.idle / total.total;
  }

  /** Retourne un pourcentage 0-100 */
  getUsage(): number {
    return Math.round(this.cpuUsage * 100);
  }

  /** Retourne le délai de pause recommandé pour les opérations bulk */
  getBulkDelay(): number {
    const usage = this.getUsage();
    if (usage < 30) return 0;        // Pleine vitesse
    if (usage < 50) return 200;      // Léger ralentissement
    if (usage < 70) return 500;      // Ralentissement marqué
    return -1;                        // -1 = PAUSE, attendre
  }
}
```

```typescript
// services/throttled-bulk.service.ts
@Injectable()
export class ThrottledBulkService {
  constructor(private cpuMonitor: CpuMonitorService) {}

  /** Attend que le CPU redescende sous 70% avant de continuer */
  async waitForCpu(): Promise<void> {
    while (this.cpuMonitor.getBulkDelay() === -1) {
      await new Promise(r => setTimeout(r, 1000));
    }
  }

  /** Pause adaptative entre chaque batch */
  async adaptivePause(): Promise<void> {
    await this.waitForCpu();
    const delay = this.cpuMonitor.getBulkDelay();
    if (delay > 0) {
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

**Utilisation dans les imports :**
```typescript
// registrations.service.ts
async bulkImport(fileBuffer: Buffer, ...) {
  const rows = await this.parseExcelInWorker(fileBuffer); // Worker thread
  const batches = chunk(rows, 50);

  for (const batch of batches) {
    await this.throttledBulk.adaptivePause(); // ⏸ Auto-throttle
    await this.prisma.bulk.registration.createMany({ data: batch });
  }
}
```

**Utilisation dans les exports PDF :**
```typescript
// badge-generation.service.ts
async generateBatchPdf(registrationIds: string[]) {
  for (const id of registrationIds) {
    await this.throttledBulk.adaptivePause(); // ⏸ Auto-throttle
    await this.generateSinglePdf(id);
  }
}
```

**Ce qui se passe en situation réelle :**
```
Client 1 : Event 5000 personnes, check-in à fond → CPU monte à 40%
Client 2 : Lance un import de 3000 lignes

Tick 1: CPU 40% → import tourne avec pause 200ms entre batches
Tick 2: CPU 55% → import ralentit, pause 500ms
Tick 3: CPU 72% → import se MET EN PAUSE ⏸
        → Les check-ins continuent normalement
Tick 4: CPU redescend à 60% → import reprend (500ms)
Tick 5: CPU 45% → import accélère (200ms)
Tick 6: Check-ins terminés → CPU 15% → import à pleine vitesse

Résultat :
  ✅ Check-ins : 0 impact, 100% disponibles
  ✅ Import : terminé, juste plus lent (2 min au lieu de 30s)
  ✅ Aucune intervention humaine nécessaire
```

#### 4. Limiter la concurrence Puppeteer (exports PDF uniquement)

**Problème :** Si plusieurs utilisateurs lancent des exports PDF en même temps → pages Chromium multiples → OOM possible.

**Solution :** Sémaphore + throttle CPU combinés :
```typescript
private readonly semaphore = new Semaphore(2); // Max 2 pages simultanées

async generateBadgePdf(registrationId: string) {
  await this.throttledBulk.adaptivePause(); // Attend si CPU > 70%
  return this.semaphore.acquire(async () => {
    // ... génération badge PDF
  });
}
```

> Note : Ceci ne concerne PAS l'impression (qui utilise du HTML direct).

#### 5. Activer pgBouncer sur Cloud SQL

Cloud SQL a un pgBouncer intégré (1 clic, 0 coût). Il mutualise les connexions côté DB.

Activer via : Console GCP → Cloud SQL → Instance → Connexions → Pooling.

Puis dans Prisma :
```
DATABASE_URL="postgresql://...?connection_limit=20&pgbouncer=true&pool_timeout=15"
```

#### 6. Configurer le container API avec des limites adaptées

```yaml
# docker-compose.prod.yml
# Si VM 2 vCPU / 4 Go :
api:
  deploy:
    resources:
      limits:
        memory: 2.5G   # Laisser 1.5 Go pour OS+Nginx+overhead
        cpus: '1.5'    # Laisser 0.5 pour OS+Nginx
      reservations:
        memory: 1G

# Si VM 4 vCPU / 8 Go :
api:
  deploy:
    resources:
      limits:
        memory: 5G     # Laisser 3 Go pour OS+Nginx+overhead
        cpus: '3.5'    # Laisser 0.5 pour OS+Nginx
      reservations:
        memory: 2G
```

#### Architecture de protection finale

```
                    Requête entrante
                         │
            ┌────────────┴────────────┐
            │                         │
     Check-in / API              Import / Export PDF
     (temps réel)                (opérations lourdes)
            │                         │
     Pool principal              Pool bulk
     (15 connexions)             (5 connexions)
            │                         │
     Aucun throttle         Throttle CPU adaptatif
     Priorité max           (pause si CPU > 30%)
            │                (stop si CPU > 70%)
            │                         │
            └────────────┬────────────┘
                         │
                    Cloud SQL
                   (+ pgBouncer)
```

#### 6. Configurer le health check pour détecter les freezes

```yaml
healthcheck:
  test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/api/health"]
  interval: 15s
  timeout: 5s
  retries: 3
  start_period: 30s
```

Si le health check échoue 3 fois → Docker redémarre le container automatiquement.

### 🟡 RECOMMANDÉS (premières semaines de production)

#### 7. Activer le monitoring applicatif

```typescript
// Métriques à exposer (Prometheus ou Cloud Monitoring)
- http_request_duration_seconds (par endpoint)
- nodejs_eventloop_lag_seconds
- nodejs_heap_used_bytes
- puppeteer_active_pages
- websocket_connections_total
- prisma_pool_active_connections
- checkin_rate_per_minute
```

#### 8. Logger les requêtes lentes

Activer le slow query logging sur Cloud SQL :
```
log_min_duration_statement = 500  # Log toute requête > 500ms
```

#### 9. Mettre en place un graceful shutdown

S'assurer que le container NestJS :
- Ferme proprement les WebSocket connexions
- Attend la fin des requêtes en cours
- Ferme Puppeteer/Chromium proprement
- Ferme le pool Prisma

#### 10. Séparer la configuration Nginx pour les WebSocket

```nginx
location /socket.io/ {
    proxy_pass http://api:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400s;  # 24h pour les WS longue durée
    proxy_send_timeout 86400s;

    # PAS de rate limiting sur les WebSockets
    # Le handshake est un seul HTTP request, ensuite c'est du WS
}
```

### 🟢 BONS À AVOIR (avant la première grosse montée en charge)

#### 11. Worker thread pour le parsing Excel

Le parsing d'un gros fichier Excel (5000 lignes) est CPU-bound et bloque l'event loop Node.js. Le déporter dans un worker thread :

```typescript
// worker/excel-parser.worker.ts
import { parentPort, workerData } from 'worker_threads';
import * as XLSX from 'xlsx';

const workbook = XLSX.read(workerData.buffer);
const rows = XLSX.utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]]);
parentPort.postMessage(rows);
```

Le thread principal (check-ins, WebSocket, API) n'est jamais bloqué par le parsing.

#### 12. Caching léger en mémoire (sans Redis)

Utiliser un cache LRU en mémoire pour les données qui changent rarement :
- EventSettings (TTL: 60s)
- BadgeTemplates (TTL: 300s)
- Roles & Permissions (TTL: 300s)

#### 13. Compression Gzip/Brotli sur Nginx

Vérifier que la compression est active pour les réponses JSON API (réduit la bande passante de 60-80%).

---

## 10. Estimation des coûts et évolution

### Phase 1 — Lancement (0–6 mois) : Infra de base

**Profil :** 3-10 tenants, events <1000 personnes, 1 event à la fois

| Composant | Spec | Coût/mois |
|-----------|------|-----------|
| Compute Engine | e2-standard-2 (2 vCPU / 4 Go) | ~40 $ |
| Cloud SQL PostgreSQL | db-custom-1-3840 (1 vCPU / 3.75 Go), 50 Go SSD | ~70–120 $ |
| Stockage SSD (VM) | 50 Go SSD persistant | ~8 $ |
| Cloud Storage (R2/GCS) | Badges PDF, backups (~10 Go) | ~5–10 $ |
| Réseau egress | ~10 Go/mois | ~1–5 $ |
| Monitoring & Logs | Basique | ~5–10 $ |
| Secret Manager | <10 secrets | ~1 $ |
| Domaine + DNS | Cloud DNS | ~1 $ |
| **TOTAL Phase 1** | | **~130–195 $/mois** |

> **Note :** Le Cloud SQL est le poste le plus cher. Si le budget est très serré au lancement, on peut temporairement héberger PostgreSQL sur la VM (et migrer vers Cloud SQL quand les revenus couvrent). Économie : ~100$/mois.
>
> **Changement clé vs. estimation initiale :** Le 2 vCPU / 4 Go est viable car l'impression de badges utilise du HTML direct (pas Puppeteer côté serveur). Économie de ~40$/mois sur le Compute Engine.

**Capacité Phase 1 :**
| Métrique | Capacité max estimée |
|----------|---------------------|
| Tenants actifs | ~20-25 |
| Users simultanés | ~50-80 |
| Events simultanés (pic check-in) | 2 |
| Attendees par event | 2000 |
| Check-ins/min (pic soutenu) | ~60-80 |
| Imports/jour | ~10 fichiers de 2000 lignes |
| WebSocket connexions | ~50-80 |
| Badges imprimés/jour | ~3000 (HTML servi, rendu sur Print Client) |

---

### Phase 2 — Croissance (6–18 mois) : Ajout cache + optimisations

**Déclencheurs :** CPU > 50% régulier, events > 2000 personnes, > 20 tenants

| Composant | Spec | Coût/mois |
|-----------|------|-----------|
| Compute Engine | e2-standard-4 (4 vCPU / 8 Go) | ~80 $ |
| Cloud SQL PostgreSQL | db-custom-2-7680 (2 vCPU / 7.5 Go), 100 Go SSD | ~150–200 $ |
| Redis (Memorystore) | 1 Go Basic | ~35 $ |
| Stockage SSD (VM) | 100 Go SSD | ~17 $ |
| Cloud Storage | ~50 Go (badges + backups) | ~10–20 $ |
| Réseau egress | ~30 Go/mois | ~5–15 $ |
| Monitoring & Logs | Étendu | ~15–25 $ |
| Secret Manager | | ~2 $ |
| **TOTAL Phase 2** | | **~315–395 $/mois** |

**Changements techniques :**
- ✅ Redis pour cache applicatif + sessions
- ✅ Pool connexions Prisma augmenté à 30
- ✅ pgBouncer activé sur Cloud SQL
- ✅ Batch imports optimisés
- ✅ Puppeteer avec sémaphore de concurrence
- ✅ Monitoring applicatif complet (Prometheus/Grafana ou Cloud Monitoring)

**Capacité Phase 2 :**
| Métrique | Capacité max estimée |
|----------|---------------------|
| Tenants actifs | ~40-50 |
| Users simultanés | ~100-150 |
| Events simultanés (pic check-in) | 3-4 |
| Attendees par event | 3000-5000 (avec badges pré-générés) |
| Check-ins/min (pic soutenu) | ~100-150 |
| Imports/jour | ~20+ fichiers |
| WebSocket connexions | ~100-150 |

---

### Phase 3 — Scale (18+ mois) : Multi-VM + Load Balancer

**Déclencheurs :** > 50 tenants, events > 5000, besoin de 0-downtime deploy, VM CPU > 60% régulier

| Composant | Spec | Coût/mois |
|-----------|------|-----------|
| Compute Engine (×2) | 2 × e2-standard-4 | ~160 $ |
| Load Balancer | HTTP(S) LB | ~20–30 $ |
| Cloud SQL PostgreSQL | db-custom-4-15360 (4 vCPU / 15 Go), 200 Go SSD, HA | ~350–450 $ |
| Redis (Memorystore) | 2 Go Standard (HA) | ~70–100 $ |
| Cloud Storage | ~100 Go | ~15–25 $ |
| Réseau egress | ~100 Go/mois | ~10–30 $ |
| CDN (Cloud CDN) | Frontend + assets | ~10–20 $ |
| Monitoring & Logs | Complet + alerting | ~25–40 $ |
| CI/CD (Cloud Build) | Builds + rolling deploy | ~5–15 $ |
| Secret Manager | | ~3 $ |
| **TOTAL Phase 3** | | **~670–915 $/mois** |

**Changements techniques :**
- ✅ 2+ VMs derrière un Load Balancer HTTP(S)
- ✅ Socket.IO Redis Adapter (WebSocket distribué)
- ✅ PrintQueue registry dans Redis (plus en mémoire)
- ✅ Badge generation sur Cloud Run workers dédiés (optionnel)
- ✅ Cloud SQL HA (failover automatique)
- ✅ CDN pour le frontend statique
- ✅ Rolling deployments (0-downtime)
- ✅ MIG (Managed Instance Group) possible pour auto-scaling

**Capacité Phase 3 :**
| Métrique | Capacité max estimée |
|----------|---------------------|
| Tenants actifs | 100+ |
| Users simultanés | 300+ |
| Events simultanés (pic check-in) | 5-10 |
| Attendees par event | 10 000+ |
| Check-ins/min (pic soutenu) | 300+ |
| WebSocket connexions | 500+ |

---

### 📊 Résumé de l'évolution des coûts

```
Phase 1 (Lancement)      ▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░  ~165 $/mois
                          → 20 tenants, 2000 attendees/event

Phase 2 (Croissance)      ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░  ~350 $/mois
                          → 50 tenants, 5000 attendees/event

Phase 3 (Scale)           ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░  ~800 $/mois
                          → 100+ tenants, 10000+ attendees/event
```

---

## Annexe : Checklist pré-production

- [ ] **VM sizing** : Provisionner e2-standard-2 (2 vCPU / 4 Go) — suffisant grâce à l'impression HTML directe
- [ ] **Rate limiting** : Ajuster les limites Nginx (30r/m est trop bas)
- [ ] **Pool DB séparé** : 2 instances Prisma (15 conn temps réel + 5 conn bulk)
- [ ] **Throttle CPU** : Implémenter CpuMonitorService + ThrottledBulkService
- [ ] **Batch imports** : `createMany()` par batch de 50 + `adaptivePause()`
- [ ] **Puppeteer** : Sémaphore (max 2 pages) + throttle CPU sur les exports PDF
- [ ] **pgBouncer** : Activer sur Cloud SQL
- [ ] **Pool DB** : Configurer `connection_limit=20` dans Prisma
- [ ] **Container limits** : Ajuster les limites Docker (5 Go RAM, 3.5 CPU)
- [ ] **Health check** : Configurer un health check Docker robuste
- [ ] **Monitoring** : Activer Cloud Monitoring avec alertes CPU/RAM/disk
- [ ] **Slow queries** : Activer le log des requêtes > 500ms sur Cloud SQL
- [ ] **Graceful shutdown** : Vérifier la fermeture propre du container
- [ ] **WebSocket Nginx** : Timeout 24h pour les connexions persistantes
- [ ] **Backups** : Configurer les backups automatiques Cloud SQL
- [ ] **SSL** : Configurer Let's Encrypt avec renouvellement auto
- [ ] **Firewall** : Autoriser uniquement 80, 443, 22 (SSH restreint)
- [ ] **Secrets** : Toutes les clés API/DB dans Secret Manager

---

> **Document généré à partir de l'analyse du code source de l'application Attendee EMS.**  
> Infrastructure cible : GCP — VM unique + Cloud SQL PostgreSQL.
