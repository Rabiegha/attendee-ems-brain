# Brief dev — Système de backup automatique de la base de données

> **Pour :** le dev en charge de l'implémentation
> **Objet :** transformer le dump **manuel** existant en un **système de sauvegarde automatique,
> fiable et testé**, avec une **vérification de restauration obligatoire en staging**.
> **Statut backlog d'origine :** [BACKLOG-TECH.md](./BACKLOG-TECH.md#L25) — « Cron backup automatique DB ».
> **Date du brief :** 30 juin 2026

---

## 0. Règle d'or (à lire en premier)

> **Une sauvegarde qu'on n'a jamais restaurée n'est PAS une sauvegarde.**
> L'objectif de ce chantier n'est pas « faire des dumps » (déjà fait), mais **prouver en continu
> qu'on peut restaurer**. Le livrable central est le **test de restore automatisé en staging**.

❌ Interdits absolus :
- Ne **jamais** restaurer un dump sur la **prod** dans ce chantier (test = **staging uniquement**).
- `pg_dump` est **read-only** : ne jamais modifier la prod ni toucher aux volumes Postgres.
- Aucune donnée PII en clair hors du VPS sans **chiffrement** (contexte RGPD/MEAE).

---

## 1. Point de départ (ce qui existe déjà)

- Script **manuel** : `attendee-ems-back/scripts/db-dump.sh`
  - dump `pg_dump -Fc` (custom, compressé) dans le conteneur `ems-postgres` ;
  - **validation** du dump côté serveur (`pg_restore --list`) **avant** de le promouvoir ;
  - copie locale + contrôle d'intégrité par comparaison de taille serveur/local.
- Guide de restauration manuel : `local-files/db-dumps/README-dump-restore.md`.

> ⚠️ Incohérence à corriger au passage : le script écrit dans `/opt/ems-attendee/backups`,
> le README documente `/opt/ems-attendee/backend/backups`. **Unifier** sur un seul chemin.

---

## 2. Ce qu'on veut ajouter (exigences)

### 2.1 Fréquence (calée sur le RPO, pas "le plus souvent possible")
> Régle : la fréquence se décide en fonction de la **perte de données max acceptable (RPO)**.
> Hors event les écritures sont faibles → inutile (et coûteux) de dumper en continu.

- **Baseline** : **1×/jour** (ex. 3 h du matin, fenêtre creuse). Suffisant vu le faible
  volume d'écritures hors event. (2×/jour acceptable si on veut une marge.)
- **Pendant un event** : **pas** de dump horaire. Un `pg_dump` en plein pic ajoute du CPU/I-O
  et tient un snapshot long pile quand le VPS est déjà saturé (goulot mesuré = CPU). À la place,
  faire 1 dump **la veille** + 1 dump **en fenêtre creuse** (nuit J1→J2) si l'event dure 2 jours.
- Le besoin de **RPO faible pendant l'event** (perte de quelques minutes max) est traité
  **hors de ce brief**, par l'**archivage WAL / PITR** décrit dans le
  [plan de continuité d'activité](../workstreams/api-scaling-lfd2026/plan-continuite-activite.md).
  Ce n'est pas un sujet de cadence de dump.
- Les dumps portent un **horodatage + raison** (déjà géré par `REASON=`).

### 2.2 Rétention (paliers cohérents avec la cadence — type grand-père/père/fils)
> Chaque palier doit correspondre à une cadence qui existe réellement.
> Comme on ne fait **qu'un dump/jour**, le palier le plus fin est le **quotidien** (pas d'horaire).

- **Quotidiens** : garder **14 jours**.
- **Hebdomadaires** : promouvoir 1 dump/semaine, garder **8 semaines**.
- **Mensuels** : promouvoir 1 dump/mois, garder **12 mois**.
- Purge automatique au-delà (et purge des `.partial` orphelins).
- Coût offsite estimé : ~80 Go total sur R2 → négligeable (~1 €/mois).

### 2.3 Copie hors-site (offsite) — **obligatoire**
- Une copie de chaque dump doit partir **hors du VPS** (un disque qui meurt ne doit pas
  emporter les backups).
- **Cible recommandée : Cloudflare R2** (déjà utilisé par le projet, cf. `r2.service.ts`).
- Le dump offsite doit être **chiffré au repos** (PII). Au minimum chiffrement R2 + accès
  restreint ; idéalement chiffrement applicatif (ex. `age`/`gpg`) avant upload.

### 2.4 Intégrité
- Conserver la validation `pg_restore --list` **avant promotion** (déjà en place).
- Vérifier la **taille** et un **checksum** (sha256) côté serveur ET après upload offsite.

### 2.5 Observabilité / alerting — **obligatoire**
- En cas d'**échec** de dump, d'upload ou du **test de restore** → **alerte** (Slack/email,
  ou « dead man's switch » type healthchecks.io : alerte si le job **n'a pas** pingé à l'heure).
- Logs horodatés conservés (succès/échec, durée, taille).

---

## 3. LE livrable central — Test de restauration automatisé en staging

> C'est la partie **non négociable**. Sans elle, le chantier n'est pas « done ».

### 3.1 Principe
Un job planifié (ex. **1×/jour**) qui :
1. récupère le **dernier dump** (serveur ou offsite) ;
2. le **restaure dans la base de staging** (`pg_restore --clean --if-exists --no-owner`) sur la
   stack **`ems-staging`** isolée (jamais la prod) ;
3. exécute des **vérifications de validité** (smoke tests) ;
4. **alerte** en cas d'échec.

### 3.2 Smoke tests minimaux après restore
- La restauration se **termine sans erreur** (code retour 0, pas d'erreurs `pg_restore`).
- **Comptages de lignes** sur les tables critiques `> 0` et **cohérents** (ex. `registration`,
  `event`, `user`, `badge`) — comparer à un ordre de grandeur attendu.
- L'**API staging démarre** et répond `200` sur `/api/health` **branchée sur la base restaurée**.
- Un **scénario lecture** clé fonctionne (ex. lister un event + ses inscriptions).
- Mesurer et **logguer la durée de restore** (= ton RTO réel : combien de temps pour repartir).

### 3.3 Critère de réussite
Le test est **vert** seulement si restore OK **et** tous les smoke tests passent. Sinon → alerte
immédiate (un backup non restaurable doit être traité comme un **incident**).

---

## 4. Implémentation attendue (orientation, pas imposée)

- **Planification** : `cron` ou (préféré) **timer systemd** sur le VPS (meilleurs logs/retries).
- **Scripts** à livrer dans `attendee-ems-back/scripts/` :
  - réutiliser/étendre `db-dump.sh` (ne pas le réécrire from scratch) ;
  - `db-backup-rotate.sh` (rétention en paliers + purge) ;
  - `db-offsite-upload.sh` (chiffrement + upload R2 + checksum) ;
  - `db-restore-staging-test.sh` (restore staging + smoke tests + alerte).
- **Secrets** : clés R2 / SSH / chiffrement via **variables d'env / secret store**, **jamais**
  commités. Principe du moindre privilège (un compte R2 dédié backups, write-only si possible).
- **Idempotence** : un job qui échoue à mi-chemin ne doit pas laisser d'état incohérent
  (fichiers `.partial`, base staging à moitié restaurée → recréer proprement).

---

## 5. Definition of Done (critères d'acceptation)

- [ ] Dumps automatiques **toutes les 6 h** + mode **horaire** activable pour un event.
- [ ] **Rétention en paliers** appliquée et purge vérifiée (pas de remplissage disque).
- [ ] **Copie offsite chiffrée** (R2) à chaque dump, avec checksum vérifié.
- [ ] **Test de restore en staging automatisé** (1×/jour) **vert**, avec smoke tests.
- [ ] **Alerting** fonctionnel sur échec dump / upload / restore (testé en provoquant un échec).
- [ ] **Runbook** mis à jour : restaurer en urgence (prod **et** local), où sont les backups,
      comment lire une alerte, RTO/RPO constatés.
- [ ] **Chemins unifiés** (corriger l'incohérence `backups/` du §1).
- [ ] Tout livré en **PR(s) séparées** et revues : 1) dump+rotation, 2) offsite, 3) test restore,
      4) alerting. Pas de commit fourre-tout.
- [ ] **Secrets** hors du repo, vérifié.

---

## 6. Garde-fous (rappel pour la review)

- Restore de test = **`ems-staging` uniquement**. Vérifier explicitement la cible (`DATABASE_URL`)
  avant tout `pg_restore` dans le script (refuser si la cible ressemble à la prod).
- `pg_dump` read-only, volumes prod intacts.
- Pas de PII en clair hors VPS (chiffrement offsite obligatoire).
- Mesurer **RPO** (perte max = intervalle entre 2 dumps) et **RTO** (durée de restore) et les
  documenter — c'est ce que la direction/le client voudra savoir.

---

## 7. Questions à clarifier avant de démarrer (à poser au lead)

1. **Cible offsite** confirmée = R2 ? (sinon S3/Scaleway).
2. **Volume DB** actuel (~2,4 Go d'après les derniers dumps) → valider que toutes les 6 h offsite
   ne sature ni le disque ni la bande passante.
3. **Canal d'alerte** voulu (Slack ? email ? healthchecks.io ?).
4. **RPO/RTO cibles** attendus par le client/la direction (ça fixe la fréquence définitive).
5. Faut-il une **rétention longue** (mensuelle, conformité) en plus des 4 semaines ?
