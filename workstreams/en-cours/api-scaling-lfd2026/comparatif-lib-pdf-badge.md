# Comparatif — librairie de rendu PDF/badge (LFD 2026)

> **Objet :** arbitrer le moteur de génération du **billet PDF / badge**, sur deux axes : **effort
> dev** et **tenue de charge le jour de l'event** (pic estimé **~3 000 demandes concurrentes**).
> Source technique détaillée : [attendee-ems-back/docs/workstreams/async-badge-email/README.md §7](../../../../attendee-ems-back/docs/workstreams/async-badge-email/README.md).
> Réf plan : [plan-action-lfd2026.md](./plan-action-lfd2026.md) Chantier B.

---

## 0. Préalable important — où tape réellement la charge

Avant de comparer, il faut **séparer deux flux** qui n'ont pas le même profil de charge :

| Flux | Déclencheur | Volume / profil | PDF concerné ? |
| --- | --- | --- | --- |
| **Inscription en ligne** | `POST /public/register` | **~12 400 inscriptions**, pic **~3 000 concurrentes** | ⚠️ **Email + QR PNG**, PAS le PDF badge aujourd'hui. Si Chantier B joint le PDF au mail → **le rendu PDF entre dans ce flux** |
| **Impression / badge sur site** | check-in / `addPrintJob` | rafales au comptoir d'accueil (jour-J physique) | ✅ rendu PDF + PNG |

> 🔑 **Conséquence de cadrage :** tant que le badge n'est **pas** joint à l'email, le rendu PDF ne
> subit **pas** les 3 000 concurrents (il est à l'impression). **Mais le Chantier B veut justement
> joindre le PDF à l'email de confirmation** → dès lors, **chaque inscription déclenche un rendu**,
> et le moteur PDF devient un facteur de charge du pic d'inscription. **C'est tout l'enjeu de ce comparatif.**
>
> Mitigation transverse (indépendante de la lib) : le rendu se fait **en worker BullMQ asynchrone**
> (hors event loop API), avec **cache R2** (un badge déjà généré n'est jamais re-rendu). Donc le vrai
> pic de rendu = **inscriptions uniques distinctes**, étalé par la file, pas 3 000 rendus simultanés.

---

## 1. Les 4 options

| # | Option | Principe | Moteur |
| --- | --- | --- | --- |
| **A** | **Garder Puppeteer, durcir** | Pool de pages borné, `page.close()` en try/finally, worker dédié, `--disable-dev-shm-usage` | Chromium complet |
| **B** | **Playwright (Chromium)** | Remplacer `puppeteer` par `playwright` (même moteur) | Chromium complet |
| **C** | **Moteur sans navigateur** | `pdf-lib` / `pdfkit` / `@react-pdf` | Rendu PDF natif (pas de HTML/CSS libre) |
| **D** | **Service de rendu externe** | Microservice headless dédié (**Gotenberg** sur **Cloud Run**, faisable sans migration GCP) | Chromium isolé hors API |

---

## 2. Comparatif effort + perf (synthèse)

| Critère | A — Puppeteer durci | B — Playwright | C — pdf-lib/pdfkit | D — Gotenberg/service |
| --- | --- | --- | --- | --- |
| **Effort dev** | 🟢 ~1 j (durcissement + worker) | 🟠 ~3–5 j (réécriture couche rendu + revalidation visuelle) | 🔴 ~1–3 sem (refonte moteur de templates) | 🟠 ~2–3 j (Cloud Run, **sans migration GCP**) |
| **Fidélité visuelle badge** | 🟢 identique (existant) | 🟢 pixel-identique attendu (même moteur) | 🔴 **perte** (ne rend pas le HTML/CSS arbitraire) | 🟢 identique (Chromium) |
| **RAM / instance** | 🔴 ~150–300 Mo (Chromium) | 🔴 ~150–300 Mo (Chromium) | 🟢 ~10–30 Mo | 🟠 déporté sur le service |
| **CPU au rendu** | 🟠 pic Chromium | 🟠 pic Chromium (≈ A) | 🟢 faible | 🟠 déporté (même coût, autre machine) |
| **Démarrage à froid** | 🟠 ~1–2 s (singleton) | 🟢 contexts isolés, mieux géré | 🟢 instantané | 🟠 dépend du service |
| **Débit sous rafale** | 🟠 limité par RAM/CPU du worker | 🟠 ≈ A, process plus stable | 🟢 élevé | 🟢 scalable (réplicas du service) |
| **Risque de régression** | 🟢 nul (on garde l'existant) | 🟠 moyen (revalidation obligatoire) | 🔴 élevé (rendu différent) | 🟠 moyen (nouvelle infra) |
| **Surface d'exploitation** | 🟢 aucune en plus | 🟢 aucune en plus | 🟢 aucune en plus | � service Cloud Run (managé, scale-to-zero — peu d'ops) |

> Échelle : 🟢 favorable · 🟠 moyen / à surveiller · 🔴 défavorable.

---

## 3. Tenue de charge le jour-J (~3 000 demandes concurrentes)

**Hypothèses de calcul (à valider en load test) :**
- Rendu badge Puppeteer/Playwright : **~0,7–2 s CPU** par badge, **~150–300 Mo** RAM/instance.
- Rendu `pdf-lib`/`pdfkit` : **~20–80 ms**, **~10–30 Mo**.
- Le rendu passe par **BullMQ** (worker, concurrence bornée) → les 3 000 demandes **ne se traduisent
  PAS** par 3 000 rendus simultanés, mais par une **file** drainée à `concurrency = N`.
- **Cache R2** : un badge déjà rendu n'est pas régénéré (idempotence).

| Scénario | A / B (Chromium) | C (sans navigateur) | D (service externe) |
| --- | --- | --- | --- |
| **Pic 3 000 concurrents, rendu en worker** | Tient **si** worker dédié + concurrence bornée (~4–8) + RAM dimensionnée ; sinon **saturation RAM/CPU = 1er point de rupture** | Tient **largement** (10× moins de RAM/CPU) | Tient **si** service répliqué ; reporte la charge ailleurs |
| **Débit soutenu** | ~quelques badges/s/worker → étalé par la file (latence acceptable car async) | ~dizaines/s | scalable horizontalement |
| **Risque de rupture jour-J** | 🟠 **RAM/CPU du VPS** (goulot mesuré actuel) | 🟢 faible | � faible si **Cloud Run** (scaling auto, charge hors VPS) |
| **Effort pour y arriver** | 🟢 faible (A) / 🟠 moyen (B) | 🔴 élevé (refonte templates) | 🟠 ~2–3 j (Cloud Run, **sans migration GCP complète**) |

> ⚠️ **Point clé honnête :** le facteur limitant jour-J n'est **pas la lib en soi**, c'est
> **l'isolation en worker + la concurrence bornée + la RAM du VPS**. Une option A bien isolée tient le
> pic. Changer de lib (B) **n'augmente pas le débit** de façon décisive (même moteur Chromium) — elle
> améliore surtout la **stabilité process**. Seule l'option C change radicalement le profil RAM/CPU,
> **au prix de la fidélité visuelle** (rédhibitoire si les templates badge restent du HTML/CSS libre).

---

## 4. Recommandation pour LFD 2026 (1 mois / 2 devs)

> ✅ **DÉCISION ACTÉE : Option D — Gotenberg sur Cloud Run** (rendu déporté hors VPS). L'analyse
> ci-dessous garde les 4 options pour la traçabilité ; l'Option A est conservée comme **fallback**
> tant que Cloud Run n'est pas validé en prod.

1. **Option A = fallback.** Le Puppeteer local **reste en place** (worker dédié, `page.close()`,
   cache R2) tant que le rendu Cloud Run n'est pas validé pixel-à-pixel. Filet de sécurité, pas la
   cible.
2. **Playwright (B) = 🔵 reporté** (workstream séparé, post-event). Même moteur → gain surtout en
   stabilité, **revalidation pixel-à-pixel obligatoire** : trop de risque pour le faire dans le mois.
3. **pdf-lib/pdfkit (C) = ❌ écarté** tant que les templates badge sont du HTML/CSS libre.
4. **✅ Service externe (D) = RETENU, même avant la migration GCP.** Cloud Run est un
   **service managé autonome** : on peut y déployer l'image **Gotenberg** seule et l'appeler en
   HTTP depuis le back, **sans migrer toute l'app**.
   - **Coût :** Gotenberg = **0 € de licence** (open source MIT). Cloud Run = **pay-per-use**
     (facturé CPU-temps/requête, scale-to-zero) → **~quelques € pour un event ponctuel**.
   - ⚠️ **Cold start** : Gotenberg embarque Chromium → 1er appel après scale-to-zero lent (~qq s).
     Mitigation : `min-instances = 1` pendant l'event (coût constant légèrement plus élevé).
   - ⚠️ **Intégration** : auth service-to-service (IAM/clé), egress VPS→GCP, ~2–3 j de dev/setup.
   - **Avantage décisif :** déporte **toute** la charge de rendu **hors du VPS** (le goulot actuel),
     sans toucher à la fidélité visuelle (même moteur Chromium).
   - **Bascule prod :** activer D quand le rendu est validé pixel-à-pixel ; sinon, le fallback A tient.

> **Garde-fou :** changement de lib **jamais en même temps** que le passage BullMQ / l'isolation
> worker (sinon impossible d'attribuer une régression). Toute bascule = **comparaison visuelle
> pixel-à-pixel** sur un jeu de templates de référence avant prod.

---

## 5. Estimation chiffrée — temps de génération des 3 000 badges *sous charge réelle*

> Analyse **ancrée sur les chiffres mesurés** du load test (campagne 25/06,
> [lfd-2026-load-test-results.md](../../../../attendee-ems-back/docs/infra/lfd-2026-load-test-results.md)),
> **PAS en isolation** : on tient compte des inscriptions, emails, logins qui tournent en même temps.

### 5.1 Ce qui est MESURÉ vs ce qui est HYPOTHÈSE

| Donnée | Source | Fiabilité |
| --- | --- | --- |
| VPS **8 cœurs / 22 Gio RAM** | load test §1 | ✅ mesuré |
| Plafond inscription **~33–37 reg/s**, **CPU-bound** | load test §4-5 | ✅ mesuré |
| **3 000 inscriptions absorbées en ~90 s** | load test TL;DR | ✅ mesuré |
| API au pic ≈ **4 cœurs saturés** (367 %/400 %) ; worker email/QR **~8 %** | load test §4.3-4.4 | ✅ mesuré |
| **Temps de rendu d'un badge Puppeteer ≈ 0,7–2 s CPU** | doc async-badge (estimation) | ⚠️ **NON mesuré → à valider S4** |
| Rendu `pdf-lib`/`pdfkit` ≈ 20–80 ms | ordre de grandeur connu | ⚠️ indicatif |

> 🔑 **Hypothèse de travail :** badge Puppeteer/Playwright = **~1,5 s CPU** (milieu de fourchette),
> **~300 Mo RAM/instance**. RAM jamais limitante (22 Gio) → **le facteur limitant est le CPU dispo**.

### 5.2 Modèle de contention (le point clé)

Deux régimes, parce que **le pic d'inscription ne dure que ~90 s** :

- **Régime 1 — pendant le pic d'inscription (~90 s) :** l'API mange ~4 cœurs, Postgres/pgBouncer/Redis
  ~1–2 → il reste **~2–3 cœurs** pour le worker badge. Le rendu badge **se met en file**, il ne faut
  **surtout pas** lui donner trop de cœurs (sinon il vole du CPU au handler d'inscription et dégrade
  la latence des 33 reg/s). → on borne la concurrence basse pendant le pic.
- **Régime 2 — après le pic (CPU libéré, ~7 cœurs) :** la file de badges **draine** à plein régime.

> ⚠️ **Insight :** le badge est **joint à un email** (pas un rendu temps réel) → personne n'a besoin du
> PDF en < 1 s. La bonne stratégie = **laisser la file drainer après le pic**, pas tout générer pendant.

### 5.3 Temps pour 3 000 badges (avec cache R2, rendus uniques)

| Option | Pendant le pic (~2–3 cœurs libres) | Drain après le pic (~7 cœurs) | Impact sur le VPS / les inscriptions |
| --- | --- | --- | --- |
| **A / B — Puppeteer/Playwright** | ~1,5–2 badges/s (concurrence bornée à 2–3) → la file s'accumule | ~4–5 badges/s (concurrence ~6–7) → **3 000 en ~12–18 min** | 🟠 consomme le CPU restant ; si concurrence trop haute **pendant** le pic → rallonge la latence inscription |
| **C — pdf-lib/pdfkit** | ~10–20 badges/s | ~50–100 badges/s → **3 000 en < 1–3 min** | 🟢 négligeable — **mais fidélité visuelle ❌** |
| **D — Cloud Run** | indépendant du VPS (autoscale) | ~30–50 instances → **3 000 en ~2–3 min** | 🟢 **zéro charge VPS** (rendu déporté) |

> **Lecture honnête :** avec l'**Option A** bien réglée (worker dédié, concurrence bornée pendant le
> pic puis relâchée), les **3 000 badges sont prêts en ~15 min après le pic**, **sans dégrader** les
> inscriptions. C'est **largement acceptable** pour un PDF envoyé par email. Le seul cas où A devient
> gênant : si le client exige le PDF **quasi-instantané** pour chacun, ou si le pic réel est **soutenu**
> (pas un burst de 90 s mais des heures pleines) → là **D (Cloud Run)** devient le bon choix.

> ⚠️ **À valider en S4 :** ajouter un **scénario badge** au load test (mesurer le vrai temps de rendu
> + l'impact concurrence sur la latence inscription). Tant que ce n'est pas mesuré, le ~1,5 s/badge
> reste une **hypothèse** — ne pas la présenter au client comme un chiffre garanti.

---

## 6. Estimation des coûts — Option D (Gotenberg sur Cloud Run)

> Gotenberg = **0 € de licence** (open source MIT). Le coût = **compute Cloud Run** (pay-per-use).
> Hypothèses : rendu ~2 s/badge, 1 vCPU + 2 Gio/instance. Ordres de grandeur (tarifs GCP indicatifs,
> région europe-west).

| Poste | Calcul | Coût event |
| --- | --- | --- |
| **Rendu des 3 000 badges** (scale-to-zero) | 3 000 × 2 s = 6 000 vCPU-s + mémoire | **≈ 0,15–0,25 €** |
| **Requêtes** | 3 000 × (0,40 $ / million) | **≈ négligeable** |
| **Egress** (PDF vers R2/back) | 3 000 × ~200 Ko ≈ 600 Mo | **≈ 0,05–0,10 €** |
| **Option : garder 1 instance « chaude »** (anti cold-start) sur 48 h d'event | 1 × 1 vCPU + 2 Gio × 48 h | **≈ 4–6 €** |

**Total réaliste pour l'event :**
- **Scale-to-zero** (accepte un cold-start de quelques s sur le 1er badge) : **< 1 €**.
- **1–2 instances chaudes** pendant les 2 jours (confort jour-J) : **~5–12 €**.

> 💡 Le coût cash est **dérisoire**. Le vrai « coût » de D = **~2–3 j de dev/setup** (déploiement
> image + auth IAM service-to-service + egress VPS→GCP + tests) — c'est ça qui le met **derrière A**
> pour le périmètre du mois, pas la facture.

---

## 7. Décision attendue

| Décision | Statut |
| --- | --- |
| **Option retenue pour l'event** | ✅ **D — Gotenberg sur Cloud Run** (rendu déporté hors VPS, faisable sans migration GCP, ~2–3 j, ~< 1–12 €) |
| Fallback conservé | **A — Puppeteer local** maintenu tant que Cloud Run n'est pas validé (bascule après comparaison pixel-à-pixel) |
| Validation par load test (3 000 concurrents simulés) | ⏳ à planifier (S4) — **inclure un scénario badge** |
| Évaluation Playwright (B) | 🔵 reporté post-event |
