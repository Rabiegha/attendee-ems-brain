# Workstream — Performance applicative du staging

## Pilotage

```text
Apps principales : attendee-ems-front, attendee-ems-back
Environnement    : staging.attendee.fr
Statut           : À faire après auth-refresh-token-hardening
Priorité         : Haute
Créé le          : 2026-07-24
Prérequis        : terminer le workstream auth-refresh-token-hardening
```

## Objectif

Réduire le temps de chargement des écrans Dashboard et Sessions en supprimant
les chargements massifs utilisés uniquement pour calculer des compteurs, puis
en remplaçant les requêtes de statistiques par session par les endpoints
agrégés déjà disponibles.

Ce chantier commence après
[`auth-refresh-token-hardening`](../auth-refresh-token-hardening/README.md).
Le diagnostic initial et le correctif réseau urgent ont été réalisés pendant
ce workstream d'authentification, mais les optimisations applicatives restent
isolées ici pour ne pas mélanger les périmètres.

## Diagnostic du 24/07/2026

Mesures réalisées vers 16 h 27, heure de Paris.

### Infrastructure écartée comme cause principale

- Front et `/api/health` répondaient en environ **30–45 ms**.
- Le VPS était peu chargé, avec environ **19 Gio disponibles** et un disque à
  **49 %**.
- PostgreSQL, pgBouncer, l'API et le worker staging étaient sains.
- Aucun OOM ni crash n'a été observé.
- L'API staging avait été recréée vers 15 h 28, vraisemblablement lors d'un
  déploiement ; son compteur de redémarrage était à zéro.

Conclusion : la latence mesurée se situe principalement dans les traitements
applicatifs et les requêtes de données, pas dans le DNS, TLS ou la capacité
instantanée du VPS.

### Requêtes lentes observées

| Requête | Temps constaté |
| --- | ---: |
| `GET /api/registrations/recent?limit=1000` | 1,9–2,15 s |
| `GET /api/events?limit=1000` | 1,7–2,1 s |
| `GET /api/attendees?page=1&limit=5...` | 1,25–1,32 s |
| `GET /api/events/:eventId/sessions/:sessionId/stats` | 472 ms en moyenne, 655 ms maximum |

Les stats de sessions représentaient **236 requêtes en une heure**.

## Causes confirmées ou fortement corrélées

### 1. Double chargement du Dashboard

Le Dashboard charge :

- 5 événements pour l'affichage, puis jusqu'à 1 000 événements pour calculer
  le total ;
- 5 inscriptions récentes pour l'affichage, puis jusqu'à 1 000 inscriptions
  pour calculer le total.

Ces seconds appels téléchargent les lignes et leurs relations Prisma alors
qu'un compteur `meta.total` existe déjà pour certains contrats, ou doit être
ajouté sans renvoyer les objets complets.

### 2. Requêtes N+1 sur les stats de sessions

La page Sessions appelle
`/events/:eventId/sessions/:sessionId/stats` individuellement. Le backend
possède déjà `/events/:eventId/sessions/stats`, conçu pour retourner un
snapshot agrégé sans N+1.

### 3. Requêtes et index PostgreSQL

Les métriques observées étaient cohérentes avec des lectures et tris
inutilement coûteux :

- cache hit PostgreSQL : **93,5 %** ;
- fichiers temporaires cumulés : **1,7 Gio** ;
- `findRecent()` charge plusieurs relations pour un maximum de 1 000
  inscriptions ;
- aucun index composite n'est actuellement adapté à la lecture récente
  filtrée par organisation et suppression, par exemple
  `registrations(org_id, deleted_at, created_at DESC)`.

L'index reste une hypothèse à valider par `EXPLAIN (ANALYZE, BUFFERS)` avant
création. Les compteurs et la réduction des volumes doivent être traités en
premier.

## Incident réseau découvert pendant le diagnostic — résolu

Le diagnostic avait initialement montré que Nginx envoyait certaines requêtes
vers `172.18.0.3` au lieu de l'API Attendee en `172.18.0.5`.

La cause exacte n'était pas une ancienne IP conservée par Nginx : les stacks
Attendee et Public-Form-Logger partageaient le réseau `ems-network` avec les
mêmes alias Compose génériques `api` et `postgres`. Le DNS Docker distribuait
donc alternativement l'alias `api` entre les deux conteneurs.

Correctif réalisé le 24/07 pendant le workstream
[`auth-refresh-token-hardening`](../auth-refresh-token-hardening/README.md) :

- création du réseau dédié `public-form-logger-network` ;
- rattachement de Public-Form-Logger et de Nginx à ce réseau ;
- retrait des conteneurs logger de `ems-network` ;
- validation puis reload à chaud de Nginx ;
- mise à jour des deux fichiers `docker-compose.prod.yml` sur le poste et le
  VPS.

Preuves après correction :

- `api` résout systématiquement vers `172.18.0.5` ;
- 30 contrôles consécutifs de l'API ont répondu `200` ;
- `/logger/health` répond `200` ;
- aucun nouveau `connection refused` observé après la bascule.

Cet incident est clos et n'entre pas dans le reste du présent chantier.

## Plan de traitement

### Lot 1 — Supprimer les surcharges évidentes

- [ ] Supprimer `useGetEventsQuery({ limit: 1000 })` du Dashboard.
- [ ] Utiliser `eventsResponse.meta.total` pour le compteur d'événements.
- [ ] Supprimer `useGetRecentRegistrationsQuery({ limit: 1000 })`.
- [ ] Faire retourner un total léger par le contrat des inscriptions récentes,
      ou utiliser un endpoint de comptage dédié.
- [ ] Vérifier que « Charger plus » continue de fonctionner.

### Lot 2 — Agréger les stats Sessions

- [ ] Exposer le contrat RTK Query de
      `/events/:eventId/sessions/stats`.
- [ ] Charger un snapshot par événement au lieu d'une requête par session.
- [ ] Conserver la requête détaillée seulement pour les données réellement
      absentes du snapshot.
- [ ] Vérifier les invalidations après création, modification, scan et undo.

### Lot 3 — Mesurer et optimiser PostgreSQL

- [ ] Mesurer les requêtes restantes avec `EXPLAIN (ANALYZE, BUFFERS)` sur
      staging.
- [ ] Réduire le `include` de `findRecent()` aux champs affichés.
- [ ] Ajouter uniquement les index justifiés par les plans d'exécution.
- [ ] Vérifier les connexions pgBouncer et les fichiers temporaires après les
      lots 1 et 2.

## Critères de sortie

- Aucun chargement `limit=1000` ne sert uniquement à produire un compteur sur
  le Dashboard.
- L'écran Sessions n'émet plus une requête de stats par session au chargement.
- Le Dashboard utile est chargé en moins de **500 ms côté API au p95** sur le
  jeu de données staging actuel.
- Le snapshot agrégé des sessions reste inférieur à **300 ms au p95**.
- Les réponses affichent les mêmes totaux qu'avant le correctif.
- Tests front/back, builds et smoke staging verts.
- Mesures avant/après consignées dans ce README.

## Hors périmètre

- Refonte visuelle du Dashboard.
- Migration générale du VPS.
- Modification des refresh tokens.
- Optimisation prématurée par ajout d'index sans plan d'exécution.

