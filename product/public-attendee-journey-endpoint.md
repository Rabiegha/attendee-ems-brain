# Endpoint public — Parcours d'un participant (Attendee Journey)

> Endpoint conçu pour qu'une application externe (scoring, gamification, dashboard
> partenaire) puisse récupérer le parcours individuel d'un participant à partir
> de la valeur encodée dans son QR code de badge.

---

## 1. Spécification

| | |
|---|---|
| **Méthode** | `GET` |
| **URL** | `/api/public/events/:publicToken/attendees/:badgeRef/journey` |
| **Auth** | Aucune — endpoint public |
| **CORS** | `Access-Control-Allow-Origin: *` (cohérent avec `/live-stats`) |
| **Tags Swagger** | `Public` |

### Paramètres d'URL

| Param | Type | Description |
|---|---|---|
| `publicToken` | string | Token public unique de l'événement (`event_settings.public_token`) |
| `badgeRef` | UUID | Valeur encodée dans le QR du badge = `registration.id` |

### Codes de réponse

| Code | Cas |
|---|---|
| `200` | Parcours retourné |
| `404` | Événement introuvable (`publicToken` invalide) **ou** participant introuvable / hors événement / soft-deleted / `badgeRef` non UUID |

> Pour des raisons de sécurité, on **ne distingue pas** "événement introuvable" de "participant introuvable" côté client : les deux renvoient `404` afin d'éviter l'énumération.

---

## 2. Schéma de réponse

```json
{
  "attendee": {
    "first_name": "Jean",
    "last_name": "Dupont",
    "company": "Acme Corp"
  },
  "totals": {
    "partners": 4
  },
  "sessions": [
    {
      "session_id": "7f3c5d2b-9b1a-4b58-9f1e-1c4a6d7e2b10",
      "session_name": "Keynote Google Cloud",
      "scanned_at": "2026-05-20T09:42:00.000Z"
    },
    {
      "session_id": "0c2d18e3-2efb-4f9a-9b34-2a6f3c1d5b88",
      "session_name": "Démo IA Générative",
      "scanned_at": "2026-05-20T10:15:00.000Z"
    }
  ]
}
```

### Sémantique des champs

| Champ | Description |
|---|---|
| `attendee.first_name` / `last_name` / `company` | Identité du participant. Provient en priorité de `attendees.*` (fiche canonique), avec fallback sur `registrations.snapshot_*` (identité figée au moment de l'inscription). `null` si non renseigné. |
| `totals.partners` | Nombre **DISTINCT** de partenaires (utilisateurs avec rôle PARTNER) ayant scanné ce participant sur cet événement. Calculé à partir de `partner_scans` (distinct sur `scanned_by`, hors scans soft-deleted). |
| `sessions[]` | Liste chronologique (ascendante) des sessions où le participant a été scanné `IN` au moins une fois. Une seule entrée par session (premier scan IN). |
| `sessions[].session_id` | UUID de la session. À utiliser côté front pour matcher avec la liste fixe d'IDs de sessions spéciales et calculer le booléen `is_special`. |
| `sessions[].session_name` | Nom de la session au moment de la lecture (non figé). |
| `sessions[].scanned_at` | Timestamp ISO-8601 UTC du **premier** scan IN. |

> ⚠️ `is_special` n'est **pas** retourné par l'API : la classification des sessions spéciales est gérée côté front à partir d'une liste fixe d'IDs. L'endpoint reste générique et réutilisable.

---

## 3. Exemple d'appel

### cURL
```bash
curl -s "https://api.attendee.example.com/api/public/events/lfd-2026-public-token/attendees/3f1e2a4b-7c91-4d8e-9b22-1a5e8c0d6f44/journey" | jq
```

### TypeScript / fetch
```ts
async function fetchAttendeeJourney(publicToken: string, badgeRef: string) {
  const r = await fetch(
    `https://api.attendee.example.com/api/public/events/${publicToken}/attendees/${badgeRef}/journey`,
  );
  if (r.status === 404) throw new Error('Event or attendee not found');
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}
```

### Calcul de score côté client
```ts
const SPECIAL_SESSION_IDS = new Set([
  '7f3c5d2b-9b1a-4b58-9f1e-1c4a6d7e2b10',
  '0c2d18e3-2efb-4f9a-9b34-2a6f3c1d5b88',
  'aa11bb22-cc33-dd44-ee55-66778899aabb',
]);

function score(journey) {
  const partners = journey.totals.partners;
  const sessionsSpecial = journey.sessions.filter((s) => SPECIAL_SESSION_IDS.has(s.session_id)).length;
  const sessionsNormal = journey.sessions.length - sessionsSpecial;
  return partners * 10 + sessionsNormal * 5 + sessionsSpecial * 20;
}
```

---

## 4. Architecture & implémentation

| Élément | Emplacement |
|---|---|
| Route | `src/modules/public/public.controller.ts` |
| Logique métier | `PublicService.getAttendeeJourney()` |
| DTO de réponse | `src/modules/public/dto/attendee-journey.dto.ts` |
| Module | `PublicModule` (déjà importé dans `AppModule`) |

### Garanties multi-tenant
1. Le `publicToken` n'est résolu **que** via `event_settings.public_token` (pas de fallback `event.id`, contrairement à `getEventByPublicToken`, pour éviter d'autoriser une recherche par UUID brute).
2. La `registration` est récupérée avec **`event_id` + `org_id`** issus de l'event résolu — cross-event impossible.
3. `partner_scans` filtrés sur `org_id + event_id + registration_id`.
4. `session_scans` filtrés par la relation `session.event_id + session.org_id`.
5. Soft-deletes respectés : `registrations.deleted_at IS NULL`, `partner_scans.deleted_at IS NULL`.

### Validation d'entrée
- `badgeRef` est validé contre une regex UUID v1-v5 avant tout accès Prisma. Un format invalide renvoie `404` (et non `400`) pour ne pas exposer la structure interne.

### Performances
- 3 requêtes Prisma maximum :
  1. `event_settings` (lookup unique sur `public_token` — indexé `@unique`)
  2. `registration` (lookup sur `(id, event_id, org_id)` — index composite existant)
  3. `partner_scans` `findMany distinct` (index `[registration_id]`)
  4. `session_scans` `findMany` filtré (index `[registration_id]`)
- Aucune jointure lourde. Réponse typiquement < 50 ms.
- Compatible avec une mise en cache HTTP courte (5-10 s) côté CDN si besoin.

---

## 5. Risques & limites

| Risque | Mitigation actuelle | Évolution possible |
|---|---|---|
| **Énumération de `badgeRef`** | UUID v4 = espace de 2^122. Pas brute-forçable. Erreur uniforme `404`. | Ajouter rate-limit IP (ex. 60 req/min) si abus constaté. |
| **Fuite d'identité (PII)** | Seulement first/last name + company (déjà imprimés sur le badge public). | Ne **jamais** ajouter `email`, `phone`, `metadata` à cette réponse. |
| **CORS ouvert** | Endpoint en lecture seule, sans cookie ni credentials. Surface d'attaque minimale. | Restreindre à l'origine de l'app externe une fois identifiée. |
| **Pic de charge LFD 2026** | Requêtes indexées, petite payload. | Cache HTTP `s-maxage=5` envisageable. |
| **Soft-deleted scans/registrations** | Exclus via `deleted_at IS NULL`. | — |
| **Sessions renommées** | Le `session_name` reflète l'état courant (non snapshot). | Acceptable : le `session_id` reste stable. |

---

## 6. Checklist QA

- [ ] `publicToken` inexistant → `404 Event not found`
- [ ] `badgeRef` non UUID → `404 Attendee not found`
- [ ] `badgeRef` d'un autre événement → `404 Attendee not found` (isolation cross-event)
- [ ] `registration.deleted_at != null` → `404 Attendee not found`
- [ ] Aucun partenaire n'a scanné → `totals.partners = 0`
- [ ] Même partenaire scanne 3 fois → `totals.partners = 1` (distinct)
- [ ] Plusieurs scans IN sur la même session → 1 seule entrée dans `sessions[]`, `scanned_at` = premier scan
- [ ] Scans `OUT` ignorés (n'apparaissent pas dans `sessions[]`)
- [ ] Header `Access-Control-Allow-Origin: *` présent
- [ ] Document Swagger visible sur `/api/docs` (tag `Public`)

---

## 7. Changelog

| Date | Auteur | Note |
|---|---|---|
| 2026-05-22 | EMS Backend | Création de l'endpoint pour usage externe (scoring LFD 2026). |
