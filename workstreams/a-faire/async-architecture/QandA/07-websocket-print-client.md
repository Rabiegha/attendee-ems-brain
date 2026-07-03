# 07 — WebSocket & Print Client

## 1. Architecture WebSocket actuelle — `FOUND_IN_CODE`

- Fichier : [src/websocket/websocket.gateway.ts](../../../../../attendee-ems-back/src/websocket/websocket.gateway.ts) (255 lignes), [src/websocket/websocket.module.ts](../../../../../attendee-ems-back/src/websocket/websocket.module.ts) (23 lignes, global).
- **Namespace** : `/events`.
- **CORS** : `origin: '*'`, `credentials: true` — ⚠️ permissif, à restreindre derrière Nginx en prod.
- **Transport** : `@nestjs/platform-socket.io` + `socket.io@4.8.3`.
- **Auth** :
  - JWT vérifié à la handshake. Token lu depuis `client.handshake.auth.token` ou header `Authorization`.
  - `JwtModule` configuré avec `jwtAccessSecret` + `jwtAccessTtl`.
- **Rooms** : org-scopées sous la forme `org:{orgId}`.

## 2. Émissions WS identifiées

| Événement | Émetteur côté backend | Cible |
|---|---|---|
| `registration:created` | `RegistrationsService.create()` (L1007) via `eventsGateway.emitRegistrationCreated()` | `org:{orgId}` |
| `registration:updated` | `RegistrationsService` (L668, 1281, 3470, 3584) | `org:{orgId}` |
| `registration:deleted` | `RegistrationsService` | `org:{orgId}` |
| `registration:checked-in` | `RegistrationsService.checkIn()` (L3218, 3336) | `org:{orgId}` |
| `event:created/updated/deleted` | `EventsService` (probable) | `org:{orgId}` |
| `session:created/updated/deleted` | `SessionsService` (probable) | `org:{orgId}` |
| `attendee:created/updated` | `AttendeesService` (probable) | `org:{orgId}` |
| `printers:updated` | `PrintQueueService.registerPrinters()` / `unregister` | `org:{orgId}` |
| `ems-client:identify` (inbound) | reçu du Print Client | callback gateway |

Méthodes du gateway :
- `emitToOrganization(orgId, event, data)` → `server.to(\`org:${orgId}\`).emit(event, data)`
- `emitToAll(event, data)` → broadcast global (rare)

## 3. Print Client — identification

- Fichier print client : `attendee-ems-print-client/` (Electron + Socket.IO côté repo séparé).
- Identification : le Print Client se connecte au namespace `/events`, envoie un message `ems-client:identify` avec `{deviceId, orgId}`.
- Le gateway stocke en mémoire : `emsClients: Map<socketId, {socketId, orgId, deviceId, connectedAt}>`.
- À la déconnexion : callback `onClientDisconnect(socketId)` exécuté par `PrintQueueService` → cleanup des imprimantes exposées.

## 4. Cycle de vie d'un print job

1. Front (ou mobile) appelle `POST /print-queue/add` avec `{registrationId, eventId, userId, badgeUrl, printerName?}`.
2. `PrintQueueService.addJob()` crée la ligne `PrintJob` en DB (status `PENDING`).
3. Le service émet (probable) un événement WS `print-job:new` (ou via `printers:updated`) ciblé sur le socket du Print Client de l'org/device.
4. Le Print Client reçoit, imprime, appelle :
   - `PATCH /print-queue/:id/printing` → status `PRINTING`
   - `PATCH /print-queue/:id/complete` → status `COMPLETED` + `duration_ms`
   - `PATCH /print-queue/:id/fail` → status `FAILED` + `error`
5. En cas de Print Client offline lors du dispatch, le job reste `PENDING` ou bascule `OFFLINE` (visible via `GET /print-queue/offline`).
6. Retry manuel : `POST /print-queue/offline/retry-all`. Dismiss : `POST /print-queue/offline/dismiss-all`.

## 5. État des imprimantes — `FOUND_IN_CODE`

- Stocké en **mémoire** : `Map<orgId, Map<deviceId, printers[]>>`.
- Source : enregistré par le Print Client via `POST /print-queue/printers`.
- Conséquences :
  - 🔴 **Reboot backend = liste vide** ⇒ tous les Print Clients doivent se ré-enregistrer.
  - 🔴 **Multi-instance backend = état désynchronisé** (sticky session indispensable, ou backend single-instance).
  - 🔴 **Workers BullMQ séparés ne sauraient pas qui est connecté**.

## 6. Auth / org-scoping WebSocket

- ✅ JWT obligatoire à la handshake (refus sinon).
- ✅ `orgId` extrait du token, room `org:{orgId}` automatique.
- ⚠️ Hypothèse à vérifier : un utilisateur multi-org choisit explicitement son `orgId` côté frontend pour le namespace ? Le code semble lier `socket.orgId` au token. À confirmer (`NEEDS_HUMAN_ANSWER` — voir [12-open-questions-for-human.md](12-open-questions-for-human.md) Q6).
- Print Client : envoie son `orgId` dans `ems-client:identify`. Hypothèse : le backend croise avec l'`orgId` du JWT pour empêcher cross-tenant — **à vérifier en lisant la méthode handler de l'événement dans le gateway**.

## 7. Risques actuels

| Risque | Niveau | Source |
|---|---|---|
| État `exposedPrinters` en mémoire | 🔴 Élevé | gateway / print-queue.service |
| CORS `*` en WS | 🟠 Moyen | gateway |
| Double impression possible si double dispatch (retry naïf) | 🟠 Moyen | absence de `unique` sur `(registration_id, status=PENDING)` ? À vérifier |
| Job perdu si Print Client se déconnecte juste après réception WS, avant `PATCH /printing` | 🟠 Moyen | flux asymétrique WS→HTTP |
| Cross-tenant si le `deviceId` est usurpé | 🟠 À vérifier | `ems-client:identify` |
| Spam WS pendant `bulkImport` (N émissions) | 🟠 Moyen | RegistrationsService |
| Pas de tampon offline d'événements WS (si Print Client offline, perd la notification de nouveau job ; il doit poll `/pending`) | 🟠 Moyen | architecture WS |

## 8. Ce qui doit rester WebSocket

- Notifications **éphémères** UI (toasts, refresh listes registrations, changement de statut event).
- Émission `printers:updated` (état coordination).
- Push de `print-job:new` au Print Client pour latence minimale (mais avec **fallback poll DB** côté client).
- `registration:checked-in` pour mettre à jour les écrans de monitoring temps réel.

## 9. Ce qui devrait passer par BullMQ

- L'**exécution réelle** de l'impression (génération badge, mise en queue, retry serveur).
- L'envoi des emails déclenchés par un statut WS (`approve` → email).
- La génération bulk de badges en arrière-plan, le Print Client recevant ensuite des notifications par chunks.

## 10. Ce qui devrait rester / passer en DB

- ✅ Statut canonique des `PrintJob` (déjà en DB).
- 🔴 **Liste des imprimantes exposées** : à déplacer en DB ou Redis (TTL court, key `printers:{orgId}:{deviceId}`) — décision à prendre selon (a) souhait scaling horizontal, (b) tolérance à la perte courte.
- 🔴 **Sessions Print Client connectées** (`emsClients`) : si on scale, sticky session ou Redis adapter Socket.IO obligatoire.

## 11. Recommandations sans encore les implémenter

1. Persister `exposedPrinters` (DB ou Redis avec TTL).
2. Adopter un **Socket.IO Redis adapter** dès qu'on prévoit > 1 instance backend.
3. Garantir l'idempotence `PrintJob` : contrainte unique sur `(registration_id, status='PENDING')` pour éviter doubles dispatch.
4. Documenter et tester le **flux de reconnexion** Print Client (re-register printers, fetch `/pending`).
5. Préparer la séparation **WS = notification** vs **BullMQ = exécution**.
