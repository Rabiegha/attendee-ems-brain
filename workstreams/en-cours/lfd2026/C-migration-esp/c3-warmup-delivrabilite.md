# C3 — Warm-up Mailgun & délivrabilité opérationnelle

> Créé le 15/07/2026 à partir de la bascule C1/C2.
> C1 = domaine/DNS/setup Mailgun. C2 = intégration applicative.
> C3 = exploitation réelle : warm-up, métriques, webhooks, qualité inbox.

## Objectif

Amener `mail.attendee.fr` à une réputation suffisante pour tenir le pic LFD 2026 sans bloquer
les inscriptions, tout en surveillant les signaux mailbox providers. Le besoin à garder en tête :
absorber des **pics autour de 3 000 emails** (vagues inscriptions / billets / notifications), pas
seulement quelques lots de test.

## État de départ

✅ Fait :

- Plan Mailgun acheté.
- Domaine EU `mail.attendee.fr` créé et vérifié.
- DNS OVH publiés : SPF, DKIM, DMARC, MX, CNAME tracking.
- Clé API Mailgun rotatée et disponible.
- Variables d'env Mailgun posées en staging et prod.
- Webhooks Mailgun créés pour staging et prod.
- Envoi testé en staging.
- Envoi testé en prod.
- Base warm-up disponible : inscrits newsletter du magasin Channel Scope.

✅ Fait le 17/07 :

- Tracking Mailgun prêt en production : `open`/`click`/`unsubscribes` actifs, protocole `https`, certificat tracking actif.
- Script opérationnel d'envoi warm-up Channel Scope (`send-warmup-wave2-channelscope.js`) avec `--limit/--offset`, throttle et journal d'envoi.
- Contenu warm-up en français confirmé (objet + preheader) avec opt-out fonctionnel (`%unsubscribe_url%`).
- Doublon unsubscribe supprimé côté script (plus d'injection CTA supplémentaire).
- Base Channel Scope validée pour J1 : ~5 220 adresses, dédupliquée, nettoyée Emailable.

## À faire

- [ ] Valider que les événements webhooks arrivent bien en base `email_events`.
- [ ] Vérifier les stats `/email/events/stats` après premiers envois.
- [x] Définir le débit initial J1 (`30s` entre envois sur le lot 100).
- [x] Mettre en place le mécanisme d'envoi warm-up (script interne piloté, pas de boucle manuelle).
- [x] Préparer le contenu warm-up Channel Scope avec opt-out clair.
- [x] Nettoyer la base newsletter avant envoi (doublons invalides traités côté source Emailable).
- [x] Segmenter de manière initiale par fournisseur destinataire (analyse domaine/top providers).
- [x] Lancer J1 en petits lots (liste interne), observer inbox/spam/bounce/complaint/deferred.
- [x] Documenter les résultats J1/J2/J3 dans ce fichier.
- [x] Ajuster J2/J3 selon signaux réels.
- [x] Programmer J4→J6b (timers systemd + `run-warmup-lot.sh`, smoke test avant chaque lot).
- [ ] Valider la **nouvelle base** avant le jeudi 23/07 ; la base Channel Scope ne sera volontairement
      pas épuisée mercredi afin de préserver une montée progressive.
- [ ] Suivre les KPIs à chaque palier (bounce/complaint/deferred/unsub) et annuler le timer suivant si signaux rouges.
- [ ] Après J7, planifier des paliers vers 1 500 / 2 000 / 3 000 emails par jour si les signaux restent propres.
- [ ] Réaliser au moins un test contrôlé proche de 3 000 emails avant l'event, avec observation queue/webhooks/providers.
- [ ] Écrire le runbook jour J : pause, baisse débit, contact support Mailgun, fallback billet hors email.

## Journal d'exécution — 2026-07-16 (J1)

- Warm-up initial sur liste interne : **30 emails**.
- Objectif: amorcer la réputation avec un lot très contrôlé avant ouverture de la base Channel Scope.

## Journal d'exécution — 2026-07-17 (J2)

- Smoke tests envoyés avec succès (adresses internes `@choyou.fr` + Gmail de contrôle).
- Vérification unsubscribe en réel :
  - désinscription effective détectée (`suppress-unsubscribe` côté events Mailgun) ;
  - réinscription test validée via suppression de l'entrée unsubscribe ;
  - nouvel envoi ensuite `delivered`.
- Custom template unsubscribe Mailgun ajusté côté dashboard (texte FR) ; correctif script appliqué pour supprimer le doublon `unsubscribe`/`Se désinscrire` dans le mail.
- Warm-up Channel Scope J2 lancé en production : lot `100` (`offset=0`, `limit=100`, `delay=30s`) avec journal CSV côté VPS.

### Message IDs de référence (trace J2)

- `20260717152919.9c3785080ad0d063@mail.attendee.fr` (smoke test)
- `20260717152923.ce3db60bc149fac0@mail.attendee.fr` (smoke test)
- `20260717155000.7af627cb5228c76e@mail.attendee.fr` (smoke test Gmail)
- `20260717155415.103d741335884afb@mail.attendee.fr` (lot 100 — envoi 1/100)

## Journal d'exécution — 2026-07-18 (J3)

- Deux envois planifiés:
  - lot 2: `offset=100`, `limit=100` (lignes 101-200)
  - lot 3: `offset=200`, `limit=150` (lignes 201-350, déclenchement auto après lot 2)
- État vérifié via le journal VPS `scripts/ops/warmup-wave2-sent-2026-07-18.csv`:
  - lot 2: **57/100** adresses Channel Scope envoyées à ce stade (et **1 smoke test** interne séparé);
  - lot 3: **0/150** à ce stade (en attente de fin du lot 2).
- Aucun échec documenté dans le journal au moment de la vérification.

### Décision de pilotage (18/07 — mise à jour)

- ~~Pause opérationnelle~~ **annulée** : on suit la rampe du plan (J3 = 250 → ligne 350 aujourd'hui).
- **Smoke test : une seule adresse** → `rgharghar@choyou.fr` (vérifié 18/07 : respecté).
- 🔄 **Changement de base le jeudi 23/07** : la base Channel Scope actuelle (5220) est utilisée
  jusqu'au **mercredi 22/07 inclus**, sans obligation de l'épuiser.
  Nouvelle base à valider avant tout envoi jeudi.
- Programmation atténuée (dimanche 19/07 = repos, base B2B) : 400 lundi, 600 mardi, puis
  900 + 600 maximum mercredi — voir [suivi-envois-channelscope.md](./suivi-envois-channelscope.md).

### Programmation posée (18/07 soir)

- **Timers systemd one-shot** : smoke à 08h puis J4/J5/J6a à 09h heure de Paris, chacun bloqué sans
  fichier `GO` créé après contrôle ; smoke J6b à 14h, aucun timer de lot J6b, lancement manuel cible
  à 15h seulement après validation.
- Les smokes passent par `scripts/ops/run-warmup-smoke.sh`, une heure avant les lots. Les lots passent
  par `scripts/ops/run-warmup-lot.sh` et exigent leur fichier `GO`. J6b ajoute un garde-fou ≥900 et
  une autorisation humaine contenant `GO 2026-07-22 2250`.
- ⚠️ Le smoke test et le garde-fou de volume valident le chemin technique, **pas la réputation**.
  Avant J6b, appliquer une validation humaine `GO / HOLD / STOP` entre 11h et 14h (heure de Paris)
  sur les signaux J6a. Le lot est lancé manuellement seulement en cas de `GO` explicite.
- Fenêtres de contrôle actées : avant chaque lot, 1-2h après, puis en fin de journée / le lendemain
  matin pour intégrer les bounces, plaintes et ralentissements tardifs.
- Accès VPS simplifié : alias SSH `ems-vps` (clé + ControlMaster, plus de mot de passe).
- Dette : `send-warmup-wave2-channelscope.js` **et** `run-warmup-lot.sh` vivent sur le VPS uniquement,
  à committer dans le repo.
- Vérification doublons 18/07 : **aucun** (dédup intra-jour OK, croisement 17/07×18/07 = seulement le
  smoke test `rgharghar@choyou.fr`, attendu).

## Leviers éventuels — à voir / détailler

- [ ] **Découpler les webhooks Mailgun de l'écriture DB directe** si le volume `opened` devient bruité :
      aujourd'hui le controller webhook persiste directement dans `email_events`. Pour LFD, ce n'est pas
      censé bloquer l'inscription si `EMAIL_QUEUE_ENABLED=true`, mais chaque email peut générer plusieurs
      événements (`accepted`, `delivered`, `opened`, réouvertures). À détailler si les métriques montrent une
      pression DB : queue `email.events`, worker dédié, batch insert, déduplication/agrégation des `opened`,
      et priorité basse par rapport aux flux d'inscription/check-in.

## Critères de succès

- Bounce hard très faible.
- Aucune plainte spam.
- Gmail/Outlook/Yahoo livrent en inbox ou au minimum sans blocage massif.
- Webhooks `accepted`, `delivered`, `temporary failure`, `permanent failure`, `spam complaints`
  visibles côté Mailgun et côté Attendee.
- Queue email stable, débit modifiable sans restart.
- Capacité validée à encaisser une vague proche de 3 000 emails sans backlog durable ni blocage provider.

## Risques spécifiques

- Utiliser une base newsletter non alignée avec le domaine Attendee peut créer des plaintes.
- Envoyer trop vite sur Gmail/Outlook au J1 peut déclencher des ralentissements.
- Un deuxième domaine d'envoi non warmé n'aide pas le jour J ; il peut même fragmenter la réputation.
- Les webhooks `opened` peuvent multiplier les écritures `email_events` après les envois ; à surveiller via
  DB/queue/API, avec le levier de découplage ci-dessus si nécessaire.
