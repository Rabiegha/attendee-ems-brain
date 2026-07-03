# 14 — Compatibilité future event-driven

> ⚠️ **OUT OF SCOPE FOR CURRENT IMPLEMENTATION**
> **BUT IMPORTANT FOR FUTURE ARCHITECTURE COMPATIBILITY**

---

## 1. Scope

Ce fichier **ne propose pas** de migrer vers une architecture event-driven complète (Pub/Sub GCP, Kafka, NATS, microservices, outbox).
Il vérifie uniquement que les choix BullMQ/Redis/workers/WebSocket envisagés à court terme **ne bloquent pas** une telle évolution future.

Points à garder en tête :
- **BullMQ sert à exécuter des jobs**, pas à publier des événements métier.
- **Domain events** décrivent des faits déjà arrivés (`registration.checked_in`, `import.completed`, etc.).
- Les deux concepts **doivent rester séparés**, même si transitoirement BullMQ peut transporter quelques événements par opportunisme.

## 2. Distinction Job vs Domain Event

### Job (command)
- Demande **d'exécuter un travail**.
- Exécutable, retryable, peut échouer.
- Exemples actuels / cibles :
  - `exports.attendees` (existant)
  - `exports.registrations` (existant)
  - `imports.processRegistrationFile` (cible)
  - `badges.generateBulk` (cible)
  - `prints.dispatchJob` (cible)
  - `emails.sendOne` / `emails.sendBatch` (cible)
  - `webhooks.n8nCheckIn` (cible)
- Naming proposé : `<domain>.<action>` (verbe à l'impératif).
- Payload : **IDs uniquement**, pas de PII ni de payload métier complet.
- Source de vérité : **DB** (`ExportJob`, `ImportBatch`, etc.).

### Domain event (fact)
- Décrit un **fait métier déjà arrivé**.
- Émis **après** un commit DB.
- Plusieurs listeners peuvent s'y abonner (email, audit, n8n, intégrations).
- Exemples cibles :
  - `imports.completed`, `imports.failed`
  - `badges.generated`, `badges.failed`
  - `prints.completed`, `prints.failed`
  - `attendees.checked_in`
  - `registrations.created`, `registrations.approved`, `registrations.refused`
  - `exports.ready`
  - `emails.sent`, `emails.failed`
- Naming proposé : `<entity>.<past_tense>`.
- Payload : minimal mais **suffisant pour audit** (`org_id`, `entity_id`, `actor_user_id`, `occurred_at`, éventuellement diff).

### Règles
- Un job peut **produire** un ou plusieurs domain events (post-commit).
- Un domain event **n'est jamais** une simple tâche BullMQ avec retry illimité ; c'est un fait — sa rediffusion est gérée par l'outbox plus tard.
- Redis/BullMQ **ne devient jamais** la source de vérité des événements métier.
- Les listeners de domain events restent **idempotents**.

## 3. Code findings (état actuel)

| Élément | État | Implication |
|---|---|---|
| `EventEmitter2` / `@OnEvent` / NestJS event bus | `NOT_FOUND` | Pas d'event bus interne. Tous les effets de bord sont câblés en dur dans les services métier. |
| Services trop couplés aux effets de bord | `FOUND_IN_CODE` | `RegistrationsService` injecte directement `EmailService`, `EventsGateway`, `N8nWebhookService`. Refacto vers « publier un event interne, listeners s'abonnent » fortement souhaitable. |
| WebSocket utilisé comme transport métier ? | `INFERRED_FROM_CODE` — partiellement. | `printers:updated` est davantage coordination que métier ; `registration:checked-in` est davantage notification UI. Pas de cas où le WS porte une décision métier irréversible — bonne nouvelle. |
| Logique métier directement liée à Redis/queue | `FOUND_IN_CODE` (limitée) | Module `Exports` est propre : payload minimal + relit DB. Pas de logique critique côté Redis. |
| Possibilité d'ajouter des domain events | OUI | Aucune contrainte technique majeure. `@nestjs/event-emitter` peut être ajouté sans refonte. |
| `AuditLog` | `NOT_FOUND` | Manque la fondation pour persister des events si on veut un futur outbox. |
| `Outbox` | `NOT_FOUND` | Aucun outbox aujourd'hui. |
| Transaction boundaries | `FOUND_IN_CODE` | Bonnes pour la DB pure (registrations.create, events.create). **Aucune émission WS/email/webhook n'est dans la même transaction que le commit Postgres** ⇒ incohérences possibles si crash entre commit et notification. C'est précisément ce qu'un outbox résoudrait. |
| Cohérence DB ↔ queue ↔ notification | À risque | Aucun mécanisme « commit-then-publish atomique ». |

## 4. Futurs blockers (choix à éviter)

| Choix | Risque | Impact futur | Comment l'éviter maintenant |
|---|---|---|---|
| Mettre toute la logique métier dans les processors BullMQ | Logique inaccessible hors job, intestable, illisible | Migration event-driven impossible sans réécrire les processors | Conserver le métier dans des **services applicatifs** ; le processor appelle le service |
| Utiliser BullMQ comme event bus métier | Fait dépendre les abonnés du retry et de la sérialisation BullMQ | Migration Pub/Sub/Kafka force réécriture | Réserver BullMQ aux **commands** ; pour les domain events, soit ne rien faire encore, soit `@nestjs/event-emitter` (in-process) |
| Ne pas créer de domain events | Couplage explicite éternel entre `Registrations` et `Email/WS/N8n/Audit` | Refactor coûteux le jour J | Au minimum, introduire `@nestjs/event-emitter` dans le pipeline des actions critiques (check-in, approve, generate badge) |
| Ne pas persister les statuts métier en DB | Source de vérité dans Redis | Toute migration event-driven nécessite repartir de la DB | **Règle d'or** : DB d'abord, queue ensuite, jobId = id DB |
| Faire confiance uniquement au payload Redis | Cross-tenant possible, désynchronisation | Migration vers outbox impossible | Processor **relit la DB** systématiquement |
| Envoyer WS/email/webhook depuis partout | Spaghetti d'effets | Difficile de migrer vers Pub/Sub | Centraliser sur des listeners (`@OnEvent('attendees.checked_in')`) |
| Pas d'idempotence | Doublons à chaque retry | Une migration vers un transport « at-least-once » (Pub/Sub) explose | Clé d'idempotence systématique (id naturel ou hash) |
| Pas de transaction boundaries claires | Incohérences DB / notif | Outbox impossible | Documenter ce qui est dans/hors tx |
| Naming d'events instable | Casse les contrats abonnés | Évolution dangereuse | Convention `<entity>.<past_tense>` versionnée |

## 5. Future-safe decisions à graver dès maintenant

1. **DB = source de vérité.** Toute action async crée d'abord la ligne (`ExportJob`, `ImportBatch`, `PrintJob`, `EmailLog`, etc.) puis enqueue avec `jobId = id DB`.
2. **Redis/BullMQ = transport de jobs**, jamais de vérité métier.
3. **Payload BullMQ minimal** : juste les IDs nécessaires pour rejouer la lecture DB côté processor.
4. **Workers rechargent depuis la DB.** Ils ne font jamais confiance à un `org_id` du payload sans contre-vérifier.
5. **Services applicatifs = logique métier**, processors = simples adapters BullMQ.
6. **Listeners pour effets de bord** : email, WS, audit, webhook n8n.
7. **Idempotence obligatoire** dans tous les handlers (clé naturelle + check DB).
8. **Naming stable** :
   - Commands : `<domain>.<action>` (ex : `imports.processRegistrationFile`).
   - Events : `<entity>.<past_tense>` (ex : `registrations.checked_in`).
9. **Domain events versionnables** : payload typé, `event_version: 1` dès le départ.
10. **Outbox-ready** : préparer `AuditLog`/`OutboxEvent` même si on ne l'exploite pas tout de suite.

## 6. Outbox pattern readiness

### Actions qui mériteraient outbox plus tard
| Action | Priorité outbox | Pourquoi |
|---|---|---|
| `imports.completed` | Haute | Beaucoup de downstream possibles (email, audit, KPIs) |
| `prints.completed` | Moyenne | Stats, audit, KPIs |
| `prints.failed` | Haute | Alerting, support |
| `badges.generated` | Moyenne | Cache invalidation, audit |
| `attendees.checked_in` | Haute | Webhook n8n + email + KPI + score affinity + WS |
| `registrations.created` (public form) | Haute | Email confirmation + admin + KPI |
| `emails.sent` / `emails.failed` | Haute | Audit, support |
| `exports.ready` | Moyenne | Email + audit |

### Ce qui est prêt
- ✅ `prisma.$transaction` utilisé sur les opérations critiques.
- ✅ `ExportJob` est une fondation correcte (DB-first pattern).
- ✅ Conventions de naming des queues centralisées (`QUEUE_NAMES`).

### Ce qui manque
- ❌ Pas d'`AuditLog`/`OutboxEvent`.
- ❌ Pas de transaction qui englobe le commit DB ET l'écriture d'un event/outbox row.
- ❌ Pas de dispatcher qui pollerait un outbox pour ré-émettre dans BullMQ/Pub/Sub.

### Ce qu'il faudra ajouter plus tard (pas maintenant)
- Table `OutboxEvent` (`id`, `aggregate_type`, `aggregate_id`, `event_name`, `event_version`, `payload Json`, `created_at`, `published_at`, `attempts`).
- Worker repeatable qui lit `OutboxEvent WHERE published_at IS NULL` et publie sur BullMQ / Pub/Sub.
- Helper `prisma.$transaction(async (tx) => { ...metier; await tx.outboxEvent.create(...); })`.

## 7. Questions humaines spécifiques (≤ 10)

1. Souhaite-t-on garder ouverte la possibilité d'un Pub/Sub GCP plus tard pour intégrations externes (CRM, marketing, analytics) ?
2. Souhaite-t-on **historiser** durablement les domain events (ex. `attendees.checked_in`) ?
3. Le check-in doit-il devenir un event métier durable utilisable par des intégrations tierces (n8n, Zapier) ?
4. A-t-on besoin de brancher des **intégrations externes** (webhook clients) plus tard ?
5. Veut-on éviter **dès maintenant** tout couplage direct `service → EmailService / EventsGateway / N8nWebhookService` (en introduisant `@nestjs/event-emitter` immédiatement) ou bien préserver le pattern actuel et ne refactorer qu'au besoin ?
6. Veut-on un **`AuditLog`** dès le Palier 1 ?
7. Veut-on poser dès le Palier 1 la convention de nommage events / queues citée en §5 ?
8. Veut-on garantir l'idempotence applicative dès le Palier 1 (clé idempotence dans tables jobs) ?
9. Veut-on imposer une **règle de revue** « pas d'`emit` WebSocket ni `sendEmail` directement dans un service métier » ?
10. Veut-on, à terme, séparer le module `print-queue` en un service à part (futur microservice) ?

## 8. Recommandation finale (synthèse)

**Ce qu'on peut faire maintenant sans implémenter event-driven** :

- ✅ Étendre BullMQ pour `imports`, `badges`, `emails`, `prints`, `webhooks n8n`, et exports manquants.
- ✅ Créer les tables manquantes (`ImportBatch`, `ImportRowError`, `EmailLog`, optionnel `EmailBatch`, optionnel `AuditLog`).
- ✅ Poser les conventions (DB-first, payload minimal, idempotence, naming).
- ✅ Refactorer les god-services (`Registrations.bulkImport`, `BadgeGeneration`, `PrintQueue` stateful).
- ✅ Introduire `@nestjs/event-emitter` pour les actions critiques (check-in, approve, badge generated) afin que les listeners deviennent l'unique chemin pour Email/WS/Webhook/Audit.

**Conventions minimales à poser dès le Palier 1** :
1. `jobId = id DB`.
2. Payload BullMQ = `{ entityId, orgId }` au strict minimum.
3. Processor relit DB, ne fait pas confiance au payload.
4. Naming `<domain>.<action>` pour queues, `<entity>.<past_tense>` pour events.
5. Idempotence garantie (contrainte unique + check DB).
6. Aucun secret/PII dans le payload BullMQ.
7. Logs avec `org_id`, `user_id`, `job_id`.

**Décisions à éviter** :
- 🚫 Mettre le métier dans les processors.
- 🚫 Utiliser BullMQ comme event bus métier (`exports.ready` consommé par 5 listeners via 5 jobs séparés).
- 🚫 Faire confiance aux payloads Redis.
- 🚫 Conserver `exposedPrinters` en mémoire après scale-out.

**Verdict** : **BullMQ peut être étendu sans bloquer une future architecture event-driven**, à condition de respecter les conventions ci-dessus dès le Palier 1. Sinon, dette technique garantie.
