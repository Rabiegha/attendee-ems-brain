# Contexte de reprise — conversation LFD 2026

> **Date :** 2026-07-19
> **But :** permettre de reprendre le travail dans une nouvelle conversation sans relire tout
> l'historique. Toujours vérifier le code et les documents vivants avant de considérer un état comme acquis.

## 1. Objectif immédiat

Ordre de travail décidé :

1. terminer la compréhension de **L9.b** ;
2. exécuter un premier test k6 ciblé sur `registerToSession` ;
3. continuer l'analyse des points de vigilance du CDC ;
4. implémenter/finaliser **C2.1** ;
5. traiter **L9.1** et les briques event critiques du chantier N ;
6. fermer le chantier O d'audit métier/sécurité LFD ;
7. exécuter le test k6 final combiné avec 3 000 clients simultanés et l'audit activé ;
8. n'ajouter Redis/cache ou envisager le multi-instance que si la mesure le justifie.

## 2. Objectifs de la semaine suivante

- finaliser le chantier B (billet PDF/Gotenberg) ;
- finaliser C2.1 (billet puis email asynchrones) ;
- avancer/finaliser les garanties event du chantier N ;
- traiter les gaps fonctionnels découverts dans le CDC ;
- préparer la suite sans démarrer les refontes post-event F/L/P avant LFD.

## 3. Décisions d'architecture actées

### Pour LFD

- une seule instance API tant que l'application n'est pas stateless ;
- inscription et check-in **synchrones, courts et atomiques dans PostgreSQL** ;
- BullMQ traite uniquement les effets secondaires après commit ;
- PDF et email deviennent asynchrones ;
- aucune impression pour LFD : PrintNode/impression est hors scope event ;
- plusieurs workers sont possibles si queue partagée, concurrence bornée et idempotence sont prouvées ;
- pas de portier Redis, cache ou multi-instance « au cas où » : décision après k6 ;
- la réservation d'une place ne doit jamais être différée dans un worker.

### Après LFD

- chantier F : rendre l'API stateless, compatible multi-instance puis HA/GCP ;
- chantier L : faire évoluer progressivement Attendee vers une architecture event-driven fiable ;
- chantier P : industrialiser après LFD l'audit et les logs techniques dans deux pipelines séparés ;
- cible L : transactional outbox/inbox, contrats versionnés, consommateurs idempotents, retry/DLQ,
  rejeu et observabilité ;
- BullMQ reste le socle envisagé tant qu'un besoin réel ne justifie pas un autre broker.

## 4. Nouveaux chantiers structurants

### N — Architecture event-ready LFD

- horizon : avant LFD ;
- estimation brute coordonnée : **~10–17,5 j-dev**, distribuée dans I/J/B/C2.1/T/K + N-ANTI ;
- ne pas additionner cette estimation une seconde fois ;
- couvre hot paths, workers PDF/email, idempotence, réconciliation, charge combinée, métriques et runbooks.

Document : [N-architecture-event-ready/README.md](./N-architecture-event-ready/README.md)

### F — API stateless, multi-instance et HA

- horizon : post-event ;
- estimation initiale : **15–27 j-dev**, environ 4–7 semaines calendaires ;
- premier jalon audit + séparation API/workers : 4–7 j-dev ;
- estimation à recalibrer après l'audit de l'état local.

Document : [F-stateless-multi-instance/README.md](./F-stateless-multi-instance/README.md)

### L — Architecture event-driven Attendee

- horizon : post-event ;
- estimation initiale : **27–46 j-dev**, environ 2–4 mois calendaires ;
- socle standards + outbox/inbox : 7–11 j-dev ;
- premier flux de référence inscription → billet → email : 10–16 j-dev, socle compris ;
- autres candidats : check-in/live, impression post-event, invitations, scoring, exports, n8n/CRM,
  webhooks, purge et intégrations.

Document : [L-architecture-event-driven/README.md](./L-architecture-event-driven/README.md)

### O — Audit métier et sécurité LFD

- horizon : avant LFD, exigence CDC ;
- estimation brute : **5–8,5 j-dev**, avec hooks partagés BIL/T/K à ne pas doubler ;
- conserve PostgreSQL `audit_logs`, mais introduit `AuditEventV1` et un port indépendant du stockage ;
- couvre login succès/échec/logout, autorisations, inscriptions, rôles, ouverture/fermeture, jauges,
  horaires et exports avec auteur/rôle snapshot, cible, résultat et avant/après ;
- `O-BIL` remplace tout sous-chantier `BIL-AUDIT` séparé.

Document : [O-audit-lfd/README.md](./O-audit-lfd/README.md)

### P — Plateforme de journalisation Attendee

- horizon : entièrement post-event ;
- estimation incrémentale après L0/L1 : **15–25 j-dev** ; autonome : **22–36 j-dev** ;
- réutilise `AuditEventV1` de O et l'outbox/inbox de L ;
- possède projections/sinks, archive, rétention, viewer org-scoped et pipeline technique séparé ;
- MongoDB est une option à mesurer, pas une décision préalable.

Document : [P-plateforme-journalisation/README.md](./P-plateforme-journalisation/README.md)

## 5. L9 et tests k6

### État connu

- L9.b `registerToSession` est livré sur staging avec `sessions.registered_count` ;
- il reste à bien comprendre le flux puis à le mesurer ;
- L9.1 concerne le check-in/compteur de présence session ;
- L9.a concerne `registerToEvent` et n'est pas la priorité immédiate si le flux LFD passe par les sessions.

### Résultat historique du 10/07

Test `register-baseline.js`, 50 VUs, 3 minutes :

- avant : 3 595/3 595 succès, p95 162 ms, médiane 60 ms, moyenne 72 ms, ~19,9 inscriptions/s ;
- après pgBouncer/pool : 3 578/3 578 succès, p95 149 ms, médiane 68 ms, moyenne 78 ms,
  ~19,8 inscriptions/s.

Interprétation : excellente stabilité et aucune régression significative, mais débit inchangé. Ce test
ne prouve ni le gain L9.b ni la conformité aux 3 000 requêtes simultanées. Environ 20 inscriptions/s
ne signifie pas que chaque requête attend : les latences individuelles restent autour de 60–150 ms.

### Campagnes décidées

1. **k6 ciblé L9.b maintenant** : débit, p95, p99, erreurs, doublons, capacité/waitlist et survente.
2. **k6 final après C2.1 + L9.1 + briques N + O** : lectures, inscriptions, check-ins, audit et
   PDF/email en arrière-plan avec 3 000 clients actifs simultanément.

Le CDC exige le résultat « 3 000 requêtes simultanées », pas Redis ou plusieurs instances. Le point
reste rouge tant que la preuve k6 n'existe pas. Si l'instance unique passe, Redis/multi-instance ne sont
pas nécessaires pour satisfaire cette exigence.

## 6. C2.1 et effets secondaires

Flux cible :

```text
inscription confirmée en DB
  -> ticket.generate
  -> worker PDF/Gotenberg
  -> stockage/statut billet
  -> email.send
  -> worker Mailgun
```

Le front reçoit rapidement la confirmation de l'inscription. Si Gotenberg ou Mailgun est indisponible,
l'inscription reste valide et les jobs peuvent être retentés/réconciliés.

Document principal :
[C2.1 — file PHP vs Attendee/BullMQ](./C-migration-esp/C2-1-email-billet-session/03-note-file-php-vs-attendee-bullmq.md)

## 7. BIL et RBAC demandé par le CDC

### Constat

Le CDC demande explicitement un accès multi-utilisateurs simultané avec trois niveaux :
administrateur, opérateur et lecteur. Il ne demande pas explicitement un éditeur libre de permissions.

`back-office-event` est authentifié via Attendee mais n'est pas role-based : tout login réussi pose une
session admin binaire. `index.php` et `save.php` ne vérifient pas les droits par action.

### Décision BIL-RBAC

- owner : **Corentin** ;
- rôles : `ADMIN_LFD`, `OPERATOR_LFD`, `READER_LFD` ;
- le MEAE peut inviter un utilisateur et lui attribuer l'un de ces rôles ;
- la matrice de permissions reste prédéfinie dans le MVP ;
- permissions limitées à l'événement LFD via scopes/EventAccess ;
- contrôles obligatoires côté serveur PHP, pas seulement des boutons cachés ;
- UI complète pour admin, opérationnelle limitée pour opérateur, lecture seule pour lecteur ;
- si `events.update` est trop large, envisager des permissions ciblées `sessions.read`,
  `sessions.operate`, `sessions.manage`.

Estimation BIL-RBAC : **2,5–5 j-dev**. Après ajout BIL-ENTREES, BIL global est réévalué à
**7–11 j-dev**, ~3 j consommés, avancement ~35 %.

Document : [BIL-billetterie/README.md](./BIL-billetterie/README.md)

## 8. SLA 99,9 %

Le CDC exige 99,9 % entre J-15 et J+1, soit environ 24 min 30 s d'indisponibilité cumulée autorisée
sur 17 jours.

Décision d'analyse :

- le CDC demande un résultat, pas explicitement une architecture HA ;
- une mesure historique externe ≥ 99,9 % est une preuve utile ;
- Netdata seul n'est pas la meilleure preuve si le serveur entier tombe ; utiliser la sonde externe
  `/api/health` comme SLI principal ;
- l'astreinte développeur et les runbooks réduisent le MTTR mais ne suppriment pas le point de panne unique ;
- reformuler le risque en orange : historique à documenter, astreinte à formaliser, risque instance
  unique à présenter/accepter ;
- inclure honnêtement l'incident prod 502 d'environ 7 minutes dans le calcul.

## 9. Anti-bot / CAPTCHA

Le CDC demande CAPTCHA ou équivalent. Le sous-chantier **N-ANTI** est maintenant rédigé dans
[N-architecture-event-ready/N-ANTI-protection-anti-bot.md](./N-architecture-event-ready/N-ANTI-protection-anti-bot.md).

MVP recommandé :

- Turnstile ou équivalent, de préférence transparent ;
- vérification du token côté serveur ;
- rate limit IP relativement haut pour ne pas bloquer un NAT ministère ;
- rate limit par email ;
- idempotency key ;
- anti-doublon et limitations métier ;
- métriques/alertes et modes `normal`, `event`, `attack` ;
- mode dégradé prévu si le fournisseur anti-bot est indisponible.

Estimation révisée après lecture du flux réel : **2,5–4,5 j-dev**. Le navigateur passe aujourd'hui par
`register.php`, puis PHP appelle Attendee ; le throttler peut donc voir une IP serveur unique pour tous
les visiteurs. N-ANTI doit trancher appel direct ou limitation au proxy avant le test k6 final.
Hors scope : fingerprint avancé, IA antifraude et WAF sur mesure.

## 10. Règles CDC et call client du 21/07

- **H-D8 ajouté** : maximum deux activités `confirmed` par date `Europe/Paris`, E2E concurrence inclus ;
- IP stricte ou signal anti-abus : porté par N-ANTI, question client ouverte ;
- identité/email du `+1` et billet/QR propre : questions client ouvertes ;
- matrice rôles/accès BIL-RBAC : validation client ouverte ;
- accès direct du MEAE au journal d'audit ou restitution sur demande, et durée de conservation ;
- lien d'invitation privé du vendredi matin ;
- validation de la fenêtre d'ouverture 30 minutes à 4 heures avant l'activité ;
- PCA/runbooks du chantier K ;
- DPA, DPO, localisation UE, conservation et suppression RGPD ;
- support/astreinte et recette complète avant ouverture.
- audit M du scan réel : queue/replay présents mais doublon session invisible, capacité non atomique
  entre participants, aucun son, cold start offline incomplet et typecheck
  mobile rouge ; M/H/L9.1/K doivent être fermés avant le GO terrain.

Document : [06-points-vigilance-cahier-charges.md](./06-points-vigilance-cahier-charges.md)

Support du call avec toutes les questions et tableau de décisions :
[08-questions-call-lfd-2026-07-21.md](./08-questions-call-lfd-2026-07-21.md)

### Tableau de bord des entrées — audit du 19/07

Le CDC demande entrées live, taux par salle/créneau et export email + présent/absent. Audit code :

- `registered_count` = réservations confirmées, jamais une preuve d'entrée ;
- le backend calcule déjà la présence actuelle (dernier scan admis = IN) et les visiteurs ayant eu
  au moins un IN admis ;
- le front Sessions les affiche partiellement, mais sans rafraîchissement après un scan mobile ;
- le WebSocket public diffuse seulement les réservations ;
- la barre actuelle utilise `session.capacity` au lieu de `checkin_capacity ?? capacity` ;
- l'export Sessions possède les emails/heures, mais exclut les confirmés non scannés de la feuille
  session et ne produit pas `Présent`/`Absent` explicitement.

Décision : pas de nouveau chantier. [J-ENTREES](./J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd)
porte snapshot/live, H/L9.1 la vérité atomique, BIL l'UI/export et K la répétition. Un participant
entré puis sorti reste `Présent` dans l'export post-event. Incrément net estimé **~1,5–3 j-dev** ;
reste event brut réévalué **~32–48,5 j-dev** pour ~30 j-dev de capacité.

## 11. Fichiers de référence

- [03-suivi-chantiers.md](./03-suivi-chantiers.md) — tableau de bord transversal ;
- [00-plan-action.md](./00-plan-action.md) — plan maître ;
- [A-I-leviers/01-suivi-leviers.md](./A-I-leviers/01-suivi-leviers.md) — suivi des leviers ;
- [A-I-leviers/03-learning-l9b-registered-count.md](./A-I-leviers/03-learning-l9b-registered-count.md) — compréhension L9.b ;
- [A-I-leviers/04-learning-reconciliation-staging-main.md](./A-I-leviers/04-learning-reconciliation-staging-main.md) — branches staging/main ;
- [M-appli-mobile-lfd2026/README.md](./M-appli-mobile-lfd2026/README.md) — audit scan mobile et lots ;
- [recette terrain scan](./M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md) — appareils, offline,
  multi-scanner et GO/NO-GO ;
- [A-I-leviers/garde-fous-deploiement-staging.md](./A-I-leviers/garde-fous-deploiement-staging.md) ;
- [06-points-vigilance-cahier-charges.md](./06-points-vigilance-cahier-charges.md) ;
- CDC original : `local-files/La Fabrique de la Diplomatie /CDC Billetterie LFD 2026.pdf` ;
- code BIL : `back-office-event/` ;
- backend : `attendee-ems-back/`.

## 12. État des modifications locales

Des changements documentaires non commités peuvent être présents dans `attendee-ems-brain`, notamment :

- création des chantiers F, L, N, O et P ;
- alignement J/C2.1/K sur inscription/check-in synchrones ;
- retrait de l'impression du scope LFD ;
- estimations F/L ;
- ajout du lot BIL-RBAC et réévaluation du chantier BIL ;
- ajout J-ENTREES/BIL-ENTREES et réévaluation du dashboard/export CDC ;
- correction des points RBAC et audit dans les vigilances CDC.

Ne pas écraser ces changements. Vérifier `git status` et `git diff` avant toute nouvelle modification.

## 13. Prompt prêt à copier dans une nouvelle conversation

```text
Nous reprenons le projet LFD 2026 dans /Users/rabiegharghar/Desktop/ems.

Lis complètement :
attendee-ems-brain/workstreams/en-cours/lfd2026/07-contexte-reprise-conversation-2026-07-19.md

Puis vérifie le git status de attendee-ems-brain sans écraser les changements locaux.
Priorité immédiate : continuer les points de vigilance du CDC, comprendre L9.b, puis préparer/exécuter
le test k6 ciblé. Respecte les décisions documentées : une API pour LFD, inscription/check-in synchrones,
BullMQ seulement après commit, impression hors scope, Redis/multi-instance uniquement si k6 le justifie.
```
