# 12 — Questions humaines (≤ 20)

> Seules les questions que le code ne peut pas trancher.
> Priorité :
> - 🔴 **BLOCKING before architecture plan** : sans réponse, l'audit final ne peut pas être figé.
> - 🟠 **IMPORTANT before implementation** : peut être tranché plus tard mais avant le code.
> - 🟢 **NICE TO HAVE** : optimisation, peut attendre.

---

## 🔴 BLOCKING

### Q1 — Volume max attendu (events, registrations, prints, imports)
- **Pourquoi** : dimensionner Cloud Run, Memorystore, Cloud SQL, concurrency workers, taille buffer XLSX, timeout, choix scale-to-zero ou pas.
- **Impact** : tous les paliers (P1 imports, P1 badges bulk, P1 prints).
- **Options si pas de réponse** : assumer 10k registrations/event max, 100 prints/min/event, imports de 5k lignes max.
- **Recommandation par défaut** : viser P95 « event de 5000 registrations, imports 10k lignes, 50 prints/min ».

### Q2 — Workers BullMQ : in-process ou containers séparés ?
- **Pourquoi** : Puppeteer + import lourd consomment beaucoup de mémoire ; cohabiter avec l'API HTTP risque OOM / latence.
- **Impact** : architecture déploiement, Dockerfiles, CI/CD.
- **Options** : (a) tout in-process Cloud Run, (b) un worker container séparé par domaine (`exports`, `imports`, `badges`, `prints`, `emails`), (c) un seul worker container générique.
- **Recommandation par défaut** : **(b)** worker séparé au moins pour `badges` (Puppeteer) et `imports`.

### Q3 — Cloud Run scale-to-zero acceptable pour les workers ?
- **Pourquoi** : un worker BullMQ scale-to-zero n'écoute pas Redis ⇒ jobs en attente jusqu'au premier trigger HTTP.
- **Impact** : coût mensuel vs UX (lag de traitement).
- **Options** : (a) min-instances=1 (coût continu), (b) Cloud Run Jobs cron pour vider la queue, (c) GKE/Compute Engine.
- **Recommandation par défaut** : **(a)** min-instances=1.

### Q4 — Stratégie de stockage : conserver R2 ou migrer Cloud Storage GCS ?
- **Pourquoi** : R2 est OVH-friendly (zéro egress), GCS est natif GCP.
- **Impact** : `R2Service` à refactorer si swap.
- **Options** : (a) garder R2, (b) GCS, (c) abstraire derrière une interface `ObjectStorageService` et choisir plus tard.
- **Recommandation par défaut** : **(a)** garder R2 + introduire interface pour préparer (c).

### Q5 — Authentification du Print Client (Electron) côté WebSocket et HTTP
- **Pourquoi** : aujourd'hui le Print Client envoie `orgId` dans `ems-client:identify`. Le backend croise-t-il avec le JWT ? Quel token utilise le Print Client (utilisateur connecté ou token machine long-lived) ?
- **Impact** : sécurité multi-tenant, GCP exposition publique du WS.
- **Options** : (a) JWT utilisateur, (b) token machine par device (`PrintDeviceToken`), (c) mTLS.
- **Recommandation par défaut** : **(b)** token machine par device pour éviter l'expiration de session utilisateur.

---

## 🟠 IMPORTANT

### Q6 — Comportement multi-org sur le WebSocket
- **Pourquoi** : un utilisateur appartient à plusieurs orgs (`OrgUser`). Le namespace `/events` rejoint-il la room de **l'org active** du JWT, ou toutes les orgs ?
- **Impact** : design des `emitToOrganization`, isolation des notifications.
- **Recommandation par défaut** : « org active du JWT » et nouveau handshake à chaque changement d'org.

### Q7 — Endpoints exports legacy (`registrations.bulkExport`, `attendees.bulkExport`)
- **Pourquoi** : redondance avec le module `Exports` BullMQ. Supprimer ou conserver ?
- **Impact** : front (potentiellement appelle l'un ou l'autre).
- **Recommandation par défaut** : déprécier puis supprimer après migration des appelants front.

### Q8 — Faut-il rejouer un import depuis le fichier source ?
- **Pourquoi** : aujourd'hui le XLSX n'est pas conservé. Si on veut un replay propre, il faut le stocker (R2).
- **Impact** : nouveau champ `ImportBatch.source_file_key`, politique de rétention RGPD.
- **Recommandation par défaut** : oui, conserver 30 j.

### Q9 — Rétention des fichiers exports (`ExportJob.file_url`)
- **Pourquoi** : R2 stocke ad vitam si non purgé ; lien signé 72 h mais fichier reste.
- **Impact** : coût, RGPD (export contient PII).
- **Recommandation par défaut** : purge automatique 30 j (cron BullMQ repeatable).

### Q10 — Stratégie retry pour `PrintJob`
- **Pourquoi** : aujourd'hui retry **manuel** uniquement. Souhaite-t-on un retry serveur automatique (e.g. 3× avec backoff) si Print Client offline > X min ?
- **Impact** : UX (pas de print perdu) vs duplication (imprimer 2 fois le même badge).
- **Recommandation par défaut** : pas de retry automatique côté serveur ; rester sur retry manuel + alerte.

### Q11 — Faut-il un `AuditLog` global (prints, exports, imports, check-ins, emails) ?
- **Pourquoi** : aucun aujourd'hui ; demandes support « qui a fait quoi » impossibles à servir précisément.
- **Impact** : nouveau modèle, listener post-job, volume DB.
- **Recommandation par défaut** : oui, modèle `AuditLog` simple, alimentation initiale par les processors les plus critiques.

### Q12 — Compatibilité avec une **future** migration event-driven (Pub/Sub, Kafka…)
- **Pourquoi** : voir [14-future-event-driven-compatibility.md](14-future-event-driven-compatibility.md). Veut-on poser dès maintenant des conventions (domain events, naming) compatibles ?
- **Impact** : décisions Palier 1.
- **Recommandation par défaut** : oui, conventions minimales sans implémenter le bus.

### Q13 — Priorité produit : prints temps réel vs imports batch ?
- **Pourquoi** : si imports tournent en arrière-plan sur le même worker, ils peuvent retarder un print urgent.
- **Impact** : choix de pools BullMQ et priorités.
- **Recommandation par défaut** : un pool dédié par domaine (cf. Q2).

### Q14 — Progression visible (UI) pour quels flux ?
- **Pourquoi** : `ExportJob.progress` existe. Veut-on la même chose pour `ImportBatch`, `BadgeGenerationJob`, `EmailBatch` ?
- **Impact** : tracking côté processor + WS push de progress.
- **Recommandation par défaut** : oui pour `ImportBatch` (long), `BadgeGenerationJob` (long), `EmailBatch` (rassurant). Non pour `EmailLog` unitaire.

### Q15 — SLA visés (prod)
- **Pourquoi** : décide entre Memorystore HA tier, Cloud SQL HA cross-zone, multi-region.
- **Impact** : coûts × 2 environ.
- **Recommandation par défaut** : 99,5 % single-region, RPO 24 h, RTO 4 h.

### Q16 — Webhook n8n : à passer en queue ?
- **Pourquoi** : aujourd'hui fire-and-forget catch-and-log dans `checkIn`. Si n8n est down, on perd l'évènement.
- **Impact** : `EmailLog`-like pour webhooks, ou une table `OutboxEvent` minimaliste.
- **Recommandation par défaut** : queue dédiée `n8n.checkin` avec retry 5×, `EmailLog`-style log.

---

## 🟢 NICE TO HAVE

### Q17 — Quelle stratégie pour Bull Board en prod ?
- **Options** : (a) Guard SUPER_ADMIN, (b) Nginx basic-auth, (c) IAP Cloud, (d) interne uniquement.
- **Recommandation par défaut** : (a) + (b).

### Q18 — Faut-il historiser les domain events (futur outbox/event store) ?
- **Recommandation par défaut** : pas maintenant ; AuditLog suffit comme base.

### Q19 — Multi-region (DR) prévu ?
- **Recommandation par défaut** : non au démarrage.

### Q20 — Adapter Socket.IO Redis dès maintenant ?
- **Pourquoi** : prérequis pour scaler horizontalement l'API.
- **Recommandation par défaut** : oui, ajout préalable à toute scale-out.
