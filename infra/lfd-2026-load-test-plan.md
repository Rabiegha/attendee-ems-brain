# LFD 2026 — Plan de tests de charge

> Document interne — Réponse technique cahier des charges client.
> Évènement diplomatique public — 4 et 5 septembre 2026.

---

## 1. Objectifs

1. Valider la capacité à absorber **3 000 requêtes simultanées** au pic d'ouverture des
   inscriptions, conformément au cahier des charges.
2. Mesurer la stabilité sur un test d'endurance d'**1 heure** à charge soutenue.
3. Vérifier l'**absence de surbooking** lorsque la jauge d'un créneau est presque pleine.
4. Mesurer la latence et le taux d'erreur sur les endpoints critiques.
5. Identifier les goulots d'étranglement (CPU, RAM, pool DB, queue PDF, queue email).

> 🔴 **Angle mort à couvrir — le PIC COMBINÉ.** Les mesures existantes portent sur l'inscription
> **isolée**. Le jour J, plusieurs flux d'**écriture** tapent **en même temps** : **inscriptions**
> (rafale, mises en file), **check-ins session** (`SessionScan`, synchrones) et **check-ins entrée**
> (`checked_in_at`, synchrones). Le vrai plafond = leur **somme**. Ajouter un scénario k6 **mixte**
> (inscriptions + scans session + scans entrée simultanés) et vérifier que le **chemin check-in
> synchrone n'est pas affamé** par la rafale d'inscriptions (isolation via pool pgBouncer réservé).
> Rappel : le check-in **ne peut pas** être mis en file (personne à la porte → réponse immédiate).
> Détail conception : [workstream 02 §2e](../workstreams/a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md).

## 2. Outil recommandé

**k6** (Grafana Labs) — déjà adapté à NestJS, scripts JavaScript/TypeScript, intégration CI
GitHub Actions, génération automatique de rapports HTML.

Alternatives :
- Artillery (YAML, plus simple, moins puissant en distributed mode).
- Locust (Python, utile si l'équipe préfère).

> Tests **OBLIGATOIREMENT** exécutés depuis un environnement **hors infrastructure cible**
> (k6 Cloud, ou VMs dédiées dans une autre région) pour mesurer la vraie latence client.

## 3. Environnements

| Env | Usage | Données |
|---|---|---|
| `staging-load` | Cible des tests | Iso‑prod (taille, config), DB anonymisée |
| `prod` | **Interdit dans la session autonome A → B0/B1** | données réelles, hors périmètre |

> La session autonome ne lance **aucune charge, même légère, en production**. Les tests de capacité
> sont réalisés sur staging/staging-load, après vérification explicite de la cible.

## 4. Métriques à mesurer

### 4.1 Côté k6 (HTTP)

| Métrique | Seuil d'acceptation |
|---|---|
| `http_req_duration` p95 | < 1 500 ms sur `/register` |
| `http_req_duration` p99 | < 3 000 ms |
| `http_req_failed` rate | < 0,1 % |
| `http_reqs` rate | ≥ 200 req/s soutenus sur 5 min |
| `iteration_duration` p95 | < 5 s (parcours complet) |

### 4.2 Côté serveur (Prometheus / Grafana / Node exporter)

| Métrique | Seuil |
|---|---|
| CPU API | < 80 % moyenne |
| RAM API | < 80 % limite container |
| Connexions Postgres actives | < 80 % pool |
| `pg_stat_activity` waiting | 0 attente > 1 s |
| Temps moyen génération badge | < 5 s p95 |
| Temps moyen envoi email | < 3 s p95 |
| Jobs en attente BullMQ (badge) | drain en < 60 s après fin test |
| Jobs en attente BullMQ (email) | drain en < 120 s après fin test |
| Erreurs Sentry nouvelles | 0 |

### 4.3 Côté métier

| Métrique | Seuil |
|---|---|
| Nombre d'inscriptions réussies | = nombre attendu |
| Nombre d'inscriptions refusées pour jauge pleine | = surplus exact (test 7) |
| **Surbooking** (inscriptions confirmées > capacité) | **0** (zéro toléré) |

## 5. Scénarios

### 5.1 T1 — Browse landing

| Champ | Valeur |
|---|---|
| Objectif | Vérifier la tenue des endpoints en lecture publique |
| Endpoints | `GET /api/public/events/:token`, `GET /api/public/events/:token/attendee-types`, `GET /api/public/events/:token/tables` |
| Données | 1 event public préparé, token fixe |
| Charge | Ramp 0 → 200 VUs sur 30 s, palier 200 VUs / 2 min |
| Seuils | p95 < 500 ms, erreur < 0,1 % |

### 5.2 T2 — Submit inscription nominal

| Champ | Valeur |
|---|---|
| Objectif | Valider l'écriture DB + queues PDF/email |
| Endpoint | `POST /api/public/events/:token/register` |
| Données | Pool de 5 000 emails uniques pré‑générés (`fixtures/emails.csv`) |
| Charge | Ramp 0 → 50 VUs sur 1 min, palier 50 VUs / 5 min |
| Seuils | p95 < 1 500 ms, erreur < 0,1 %, badges générés < 60 s |

### 5.3 T3 — Pic 3 000 simultanés (preuve technique ; contrat à clarifier)

> ⚠️ Le squelette historique ci-dessous maintient 3 000 VUs en boucle pendant 60 secondes et réutilise
> un email par VU. Il ne prouve donc pas à lui seul « 3 000 personnes uniques, une inscription
> chacune ». Avant exécution, le séparer en un scénario one-shot de 3 000 identités uniques et des
> scénarios `ramping-arrival-rate` pour mesurer le débit. Appliquer le
> [protocole de validation du générateur](../workstreams/en-cours/lfd2026/session-travail-autonome/specs/2026-07-20-protocole-generateur-k6.md).

| Champ | Valeur |
|---|---|
| Objectif | Démontrer la tenue du seuil contractuel |
| Endpoints | `GET /api/public/events/:token` puis `POST /api/public/events/:token/register` |
| Données | 3 000 emails uniques |
| Charge | Ramp 0 → 3 000 VUs sur 10 s, palier 3 000 VUs / 60 s, ramp down 60 s |
| Seuils | p95 < 2 500 ms (toléré sur pic), erreur < 1 %, **0 surbooking** |
| Critère succès | ≥ 95 % des inscriptions terminées en succès, drain queues < 5 min |

### 5.4 T4 — Endurance 1 h

| Champ | Valeur |
|---|---|
| Objectif | Détecter fuites mémoire, dérives latence, saturation queues |
| Endpoints | Mix : 70 % browse, 25 % register, 5 % live‑stats |
| Charge | Palier constant 100 VUs / 60 min |
| Seuils | p95 stable ± 20 %, RAM API stable, pas de leak Puppeteer |

### 5.5 T5 — Spike 09:00 / 11:00 / 14:00

| Champ | Valeur |
|---|---|
| Objectif | Reproduire le pattern d'ouverture de créneau |
| Endpoint | `POST /api/public/events/:token/register?slot=09:30-12:00` |
| Charge | 0 → 1 500 VUs en 5 s, palier 30 s, ramp down 30 s, répété 3 fois (simule 09 h, 11 h, 14 h) |
| Seuils | p95 < 2 000 ms, file d'attente drainée < 2 min, 0 surbooking |

### 5.6 T6 — Contention jauge presque pleine

| Champ | Valeur |
|---|---|
| Objectif | Vérifier le verrou transactionnel sur capacité |
| Endpoint | `POST /api/public/events/:token/register` |
| Setup | Créer une session avec capacité = 100, pré‑remplir à 95 |
| Charge | 200 VUs en parallèle tentent d'inscrire au même créneau |
| Seuils | Exactement 5 inscriptions confirmées, 195 refusées en `409 Conflict` ou `waitlist` |

### 5.7 T7 — Anti‑surbooking (test concurrentiel pur)

| Champ | Valeur |
|---|---|
| Objectif | Garantir l'unicité de capacité même sous race condition |
| Setup | Capacité = 1, pré‑rempli à 0 |
| Charge | 50 VUs envoient simultanément la même requête |
| Seuils | **1 succès, 49 refus exactement.** Aucun test ne doit jamais passer avec ≥ 2 succès. |

### 5.8 T8 — Check‑in massif

| Champ | Valeur |
|---|---|
| Objectif | Valider le débit scan jour J |
| Endpoint | `POST /partner-scans` |
| Setup | 5 000 registrations valides + JWT scanner |
| Charge | 50 VUs / 5 min, soit ~ 500 scans/min |
| Seuils | p95 < 600 ms, erreur < 0,1 %, pas de double check‑in |

### 5.9 T9 — Print job creation

| Champ | Valeur |
|---|---|
| Objectif | Vérifier la création de print jobs en pic |
| Endpoint | endpoint print queue (à confirmer dans `src/print-queue/`) |
| Charge | 20 VUs / 2 min, ~ 50 jobs/min |
| Seuils | Tous les jobs créés, WebSocket `printers:updated` reçu < 2 s |

### 5.10 T10 — Mixte jour J réaliste

| Champ | Valeur |
|---|---|
| Objectif | Simuler une journée d'évènement complète accélérée |
| Charge | 60 min : 30 % register + 50 % scan + 20 % print + dashboard WS connectés (50 sessions) |
| Seuils | Tous seuils ci‑dessus tenus simultanément |

## 6. Commandes exemples

> Squelettes — à coder dans `tests/load/` au moment de l'implémentation.

### k6 — T3 pic 3 000 simultanés

```javascript
// tests/load/t3-spike-3000.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';

const emails = new SharedArray('emails', () =>
  JSON.parse(open('./fixtures/emails-3000.json'))
);

export const options = {
  stages: [
    { duration: '10s', target: 3000 },
    { duration: '60s', target: 3000 },
    { duration: '60s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<2500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const token = __ENV.PUBLIC_TOKEN;
  http.get(`${__ENV.BASE_URL}/api/public/events/${token}`);
  const i = __VU - 1;
  const res = http.post(
    `${__ENV.BASE_URL}/api/public/events/${token}/register`,
    JSON.stringify({
      first_name: 'Test',
      last_name: `User${i}`,
      email: emails[i % emails.length],
      attendee_type_id: __ENV.ATTENDEE_TYPE_ID,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  check(res, { 'status is 201': (r) => r.status === 201 });
  sleep(1);
}
```

```bash
# Exécution
BASE_URL=https://staging-load.example PUBLIC_TOKEN=xxx \
  k6 run tests/load/t3-spike-3000.js --out json=results/t3.json
```

### k6 — T7 anti‑surbooking

```javascript
// tests/load/t7-overbook.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    burst: { executor: 'per-vu-iterations', vus: 50, iterations: 1, maxDuration: '10s' },
  },
};

export default function () {
  const res = http.post(
    `${__ENV.BASE_URL}/api/public/events/${__ENV.TOKEN}/register`,
    JSON.stringify({
      first_name: 'Race',
      last_name: `${__VU}`,
      email: `race-${__VU}-${Date.now()}@test.local`,
      attendee_type_id: __ENV.ATTENDEE_TYPE_ID,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  check(res, {
    'status 201 or 409': (r) => r.status === 201 || r.status === 409,
  });
}
```

Vérification post‑test (psql) :
```sql
SELECT COUNT(*) FROM registrations
WHERE event_attendee_type_id = '<id>' AND status = 'approved';
-- Doit être EXACTEMENT 1
```

### k6 — T4 endurance

```bash
k6 run --vus 100 --duration 60m tests/load/t4-endurance.js \
  --out experimental-prometheus-rw
```

## 7. Calendrier des tests

| Date | Test | Critère go/no‑go |
|---|---|---|
| J‑20 | T1, T2 | Baseline, pas de régression |
| J‑15 | T6, T7 | Verrou capacité OK |
| J‑12 | T8, T9 | Scan + print OK |
| **J‑10** | **T3 (3 000 simultanés)** | **Critère contractuel** |
| J‑8 | T5 (spike 09 h/11 h/14 h) | OK |
| J‑7 | T4 endurance 1 h | OK |
| J‑5 | T10 mixte | OK |
| J‑3 | Re‑run T3 + T4 | Validation finale |

Tout test non go déclenche un **plan de remédiation** avec re‑run obligatoire avant J‑3.

## 8. Livrables tests

- Script k6 versionné dans `tests/load/`.
- Rapports HTML k6 + dump Prometheus, archivés dans R2 `lfd-2026/load-tests/`.
- Synthèse Markdown 2 pages avec verdict go/no‑go remise au client.
- Captures Grafana de chaque test.

## 9. Informations à confirmer

- Disponibilité d'un environnement `staging-load` iso‑prod.
- Provider SMTP de test (sandbox) pour éviter d'envoyer des emails réels.
- Accès Grafana / Prometheus partagé avec le client (optionnel).
- Acceptation du client pour la fenêtre de tests (impact réseau, alertes).
