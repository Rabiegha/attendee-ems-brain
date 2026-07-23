# Suivi des envois warm-up — base Channel Scope

> **À QUOI SERT CE FICHIER :** historique détaillé pour savoir **exactement quelles lignes de la
> base ont été envoyées quel jour**. Avant ce fichier, on ne pouvait retrouver l'historique qu'en
> fouillant les journaux CSV sur le VPS — ce qui a créé une confusion (J2 vs J3) le 18/07.
> La **source de vérité unique** pour la stratégie, l'état réel et les cumuls est désormais
> [warm-up-strategie.md](./warm-up-strategie.md). La mettre à jour en premier après chaque envoi,
> puis reporter ici le détail du lot.
>
> Stratégie / paliers cibles → [warm-up-strategie.md](./warm-up-strategie.md)
> Détails techniques / risques → [c3-warmup-delivrabilite.md](./c3-warmup-delivrabilite.md)

## Préparation de la campagne des 23 et 24 juillet

État au 23/07/2026 à 10h30 Paris :

- nouvelle newsletter : `local-files/warm-up/NL-23-24.html`, copiée sur le VPS dans
  `/tmp/NL-23-24.html` ;
- nouvelle liste : 1 320 adresses valides et uniques, copiée dans
  `/tmp/list-attendees-test-pour-envoi-23-et-24-juillet-2026.csv` ;
- lot du 23/07 : offset `0`, limite `600`, délai de base `15s` ;
- lot du 24/07 : offset `600`, limite `720`, délai de base `15s` ;
- les dry-runs vérifient exactement 600 puis 720 destinataires ;
- le runner et le script d'envoi exigent tous deux le GO exact. Pour le 24/07 :
  `GO 2026-07-24 600` dans `/tmp/warmup-go-2026-07-24-600` ;
- sans ce fichier exact au déclenchement, le lot s'annule avant le premier destinataire ;
- timers VPS actifs pour le 24/07 : smoke à 06h00 UTC / 08h00 Paris, lot à 07h00 UTC / 09h00
  Paris. Le fichier de GO est absent au moment de la programmation ;
- smoke du 23/07 : livré et ouvert sur `rgharghar@choyou.fr`; livré sur `brais@choyou.fr`.
  `brais` a reçu deux smokes au lieu d'un, car une session distante supposée interrompue a repris
  trois secondes après la relance ciblée. Un verrou inter-processus est désormais actif ;
- `GO 2026-07-23 0` reçu ; lot de 600 lancé à 10h29 Paris. Contrôle initial : trois messages
  acceptés, premier message livré, aucune erreur. Le bilan final sera inscrit à la fin du lot.

## État vérifié — 22/07/2026 à 16h38 Paris

Réponse courte pour l'exploitation :

- **Hier, mardi 21/07 : OUI, le warm-up a été fait.** Le lot J5 a envoyé **600/600 newsletters**
  entre **10h52 et 13h23 Paris** (08h52→11h23 UTC). Le log d'exécution contient 600 succès avec
  leurs `messageId` Mailgun et se termine avec `exit 0`. Ici, « envoyé » signifie **accepté par
  l'API Mailgun avec un messageId** ; le bilan final delivered/failed/complained reste à consolider.
- **Aujourd'hui, mercredi 22/07 : aucun lot de masse n'a été envoyé.** J6a (900 prévus) s'est arrêté
  à 09h Paris avant envoi, faute du fichier `GO 2026-07-22 1350`. J6b n'a pas été lancé. Aucun
  processus warm-up et aucun timer warm-up ne sont encore actifs à 16h38.
- **Deux smoke tests seulement ont été envoyés aujourd'hui**, à `rgharghar@choyou.fr`, à **08h00**
  et **14h00 Paris**. Ce sont donc 2 messages, mais une seule adresse interne, pas un lot Channel Scope.
- **Cumul newsletter au 22/07 16h38 : 1 300 envois de lots + 12 tests/smokes = 1 312 messages vers
  1 303 adresses uniques.** Les 30 emails internes de J1 ne sont pas comptés ici car ils utilisaient
  le contenu warm-up interne, pas la newsletter Channel Scope.
- **Réconciliation Mailgun :** le compteur **Sent = 1 458** affiché dans le tableau de bord porte sur
  tous les emails du domaine et pas uniquement sur cette newsletter. Le filtrage des événements
  Mailgun `accepted` sur l'objet exact `L'actualité channel de la semaine` retourne **1 312
  newsletters**. Les compteurs globaux `Delivered = 1 408`, `Failed = 94` et `Opened = 826` ne doivent
  donc pas être attribués tels quels à la newsletter.

Listes consolidées :

- [CSV détaillé — un enregistrement par envoi](./destinataires-nl-warmup-au-2026-07-22.csv)
- [TXT — adresses uniques](./destinataires-nl-warmup-uniques-au-2026-07-22.txt)

### Fiabilité des preuves

- **Source principale du cumul : événements Mailgun `accepted` filtrés par objet exact.** Cet audit a
  retrouvé **7 tests du 17/07** qui n'étaient pas consignés dans les journaux locaux, ce qui explique
  pourquoi le précédent total de 1 305 était faux.
- J2 à J4 : plages de la base source confirmées par les suivis d'exécution, tous les envois marqués OK.
- J5 : les **600 adresses du log d'exécution correspondent exactement** aux lignes 751–1350 de la
  base normalisée. Le CSV actuellement présent sur le VPS ne contient que les succès 16 à 600
  (585 lignes, sans en-tête) ; le log complet fait foi pour les 15 premiers et expose leurs
  `messageId`. Cette anomalie de journal doit être corrigée avant un prochain lot.
- 22/07 : les deux smoke tests sont présents dans le journal VPS avec horodatage et `messageId`.

## Décisions opérationnelles (18/07/2026)

- ~~Pause des envois~~ **ANNULÉE** — on suit le plan de warm-up (J3 = 250 envois aujourd'hui, jusqu'à la ligne 350).
- **Smoke test : une seule adresse autorisée** → `rgharghar@choyou.fr` (vérifié le 18/07 : respecté — 1 seul smoke test envoyé, à cette adresse).
- 🔄 **La base utilisée change JEUDI 23/07** : la base Channel Scope actuelle (5220 adresses) n'est utilisée
  que jusqu'au **mercredi 22/07 inclus**. À partir du jeudi 23/07 → nouvelle base (à définir/valider avant envoi).
- La newsletter `NL_16 juillet 2026.html` suit le même calendrier (dernier jour d'utilisation : 22/07).
- **Contrôles obligatoires** : smoke test une heure avant chaque lot, validation visuelle et Mailgun,
  puis contrôles 1-2h après et en fin de journée / le lendemain matin. Un smoke test confirme le
  chemin technique et le rendu ; il ne valide pas à lui seul la réputation du domaine.
- **J6b sous validation humaine** : le lot de mercredi après-midi ne part que sur décision explicite
  `GO` après analyse de J6a. Le simple garde-fou « ≥ 900 lignes journalisées » est insuffisant.

## Décisions opérationnelles (21/07/2026)

- ⚠️ **Le fichier `GO` n'a été créé à temps ni le 20/07 (J4) ni le 21/07 (J5)** avant le déclenchement
  du timer de 07h00 UTC (09h Paris) → les deux lots automatiques se sont **annulés d'eux-mêmes**
  (garde-fou fonctionnel, pas un bug). Rappel : le `GO` doit être écrit à Codex (*« GO J5 »* etc.) ou
  créé en SSH **avant 09h Paris**, sinon le lot du jour ne part pas tout seul.
- **J4 (20/07)** : relancé **manuellement** après coup (07h53→10h47 UTC), mais avec `limit=350` au lieu
  des `400` prévus → seules les lignes **351–700** ont été envoyées (au lieu de 351–750). **Écart de
  50 adresses jamais envoyées, lignes 701–750** (`clara.lakhdari@axis.com` → `conteg@conteg.cz`).
- **Signaux Mailgun J4 vérifiés avant de valider J5** (cumul depuis le 20/07 07h50) : 383 accepted,
  377 delivered, **207 opened (~55 %, très bon signal)**, 13 failed (8 permanents/2,3 %, 5
  temporaires/1,4 %, dispersés sur 12 domaines), **0 plainte spam** → signal 🟢 **VERT**.
- **J5 (21/07)** : décision prise de **ne pas rattraper l'écart J4** (option B) — le lot du jour garde
  `offset=750, limit=600` tel que programmé initialement. Les 50 adresses lignes 701–750 restent
  définitivement non envoyées sur cette base (acceptable : le plan ne vise pas l'épuisement de la
  base, cf. [warm-up-strategie.md](./warm-up-strategie.md)). `GO 2026-07-21 750` créé manuellement et
  lot relancé à **08:52 UTC** (smoke du matin 06:00 UTC déjà OK, non répété). **Résultat final :
  600/600 envoyés, fin à 11:23 UTC, exit 0.**

## Décisions opérationnelles (22/07/2026)

- **J6a non envoyé** : timer exécuté à 07:00 UTC, arrêt immédiat aux pré-contrôles car le fichier
  `/tmp/warmup-go-2026-07-22-1350` était absent. Volume du lot : **0/900**.
- **J6b non envoyé** : aucun lancement manuel effectué. Volume du lot : **0/600**.
- Les smoke tests de 06:00 UTC et 12:00 UTC sont partis malgré l'absence de lots ; ils sont tous deux
  journalisés sur `rgharghar@choyou.fr`.
- **Ne pas lancer rétroactivement J6a/J6b sans une nouvelle décision explicite**, une vérification
  Mailgun consolidée et une base validée : l'ancienne base et cette newsletter arrivent en fin de
  fenêtre d'utilisation le 22/07.

## Il n'y a qu'un seul script d'envoi — pas un script par jour

Un unique script gère tous les envois, tous les jours :

```
/opt/ems-attendee/backend/scripts/ops/send-warmup-wave2-channelscope.js
```

Depuis le 18/07 s'ajoute un **runner de programmation** (smoke test + garde-fou + lot) utilisé par
les timers systemd :

```
/opt/ems-attendee/backend/scripts/ops/run-warmup-lot.sh
```

⚠️ Ces scripts **ne sont pas commités dans le repo** — ils n'existent que sur le VPS (dette identifiée, à
committer un jour). Seuls les paramètres changent d'un jour à l'autre (`--offset`, `--limit`), jamais
le script lui-même. Les noms comme "J2"/"J3" dans d'anciens noms de fichiers de log étaient des
étiquettes ad-hoc de session, **pas une convention du projet** — à ne plus utiliser en dehors de ce
tableau.

Fichiers sources utilisés (chemins réels sur le VPS, différents du chemin par défaut du script) :

- Liste : `/tmp/liste-channelscope-emailable.csv` (5220 adresses, dédupliquées/nettoyées Emailable)
- Contenu : `/tmp/NL_16 juillet 2026.html` (newsletter réelle Channel Scope)
- Journal quotidien (auto, **append** — jamais écrasé) : `scripts/ops/warmup-wave2-sent-YYYY-MM-DD.csv`

## Commande type pour un nouveau lot

```bash
node -r dotenv/config scripts/ops/send-warmup-wave2-channelscope.js dotenv_config_path=.env.production \
  --send --csv /tmp/liste-channelscope-emailable.csv --html "/tmp/NL_16 juillet 2026.html" \
  --offset <PROCHAINE_LIGNE_NON_ENVOYEE> --limit <NB_DU_JOUR> --delay 30 \
  --from "ChannelScope <no-reply@mail.attendee.fr>"
```

**Règle d'or avant de lancer un lot :** `--offset` = ligne juste après la dernière ligne du tableau
ci-dessous (colonne "Jusqu'à ligne"). Ne jamais deviner — toujours lire ce tableau d'abord.

## Tableau de suivi détaillé

| Date       | Jour | Lot | Lignes CSV (offset→+limit)              | Jusqu'à ligne | Ciblé | Envoyés OK                                                                                | Échecs       | Statut      | Signaux observés                                                                     |
| ---------- | ---- | --- | --------------------------------------- | ------------- | ----- | ----------------------------------------------------------------------------------------- | ------------ | ----------- | ------------------------------------------------------------------------------------ |
| 2026-07-16 | J1   | —   | Liste interne (hors base Channel Scope) | —             | 30    | 30                                                                                        | 0            | ✅ Terminé  | Liste interne uniquement (warm-up initial)                                           |
| 2026-07-17 | J2   | 1   | 1–100 (offset=0, limit=100)             | **100**       | 100   | 100                                                                                       | 0            | ✅ Terminé  | Propre — pas d'alerte documentée                                                     |
| 2026-07-18 | J3   | 2   | 101–200 (offset=100, limit=100)         | **200**       | 100   | 100/100 (journal `warmup-wave2-sent-2026-07-18.csv`, +1 smoke test `rgharghar@choyou.fr`) | 0            | ✅ Terminé  | Smoke test 18/07 : accepted→delivered→opened en ~5s, propre. Aucun doublon (vérifié) |
| 2026-07-18 | J3   | 3   | 201–350 (offset=200, limit=150)         | **350**       | 150   | 150/150 (journal `warmup-wave2-sent-2026-07-18.csv`)                                       | 0            | ✅ Terminé  | Rythme ~1 email/30s. Aucun doublon vérifié                                            |
| 2026-07-20 | J4   | 4   | 351–700 (offset=350, **limit=350 réel** — 400 prévu) | **700** (750 prévu) | 400 prévu / 350 réel | 350/350 (journal `warmup-wave2-sent-2026-07-20.csv`, +1 smoke `rgharghar@choyou.fr` 06:00 UTC) | 8 permanents + 5 temporaires (13, cumul J4+smoke J5) | ✅ Terminé (hors mécanisme `GO`) | Timer auto 07:00 UTC **annulé** (fichier `GO` absent). Lot relancé **manuellement** 07:53→10:47 UTC avec `limit` réduit à 350 → écart lignes 701–750 (50 adresses) jamais envoyées. 0 plainte spam, ~55 % d'ouverture |
| 2026-07-21 | J5   | 5   | 751–1350 (offset=750, limit=600)        | **1350**      | 600   | **600/600** (log complet 08:52→11:23 UTC, exit 0)                                           | 0 à l'envoi  | ✅ Terminé | Timer auto 07:00 UTC annulé faute de GO, puis relance manuelle à 08:52 UTC avec `skip_smoke=1`. Smoke 06:00 UTC OK. Les 600 destinataires du log correspondent exactement aux lignes 751–1350. Écart J4 (701–750) non rattrapé. |
| 2026-07-22 | J6a  | —   | 1351–2250 (offset=1350, limit=900)      | **1350 inchangé** | 900 | **0/900**                                                                                  | —            | ⛔ Non lancé | Timer 07:00 UTC arrêté aux pré-contrôles : fichier GO absent. Smoke 06:00 UTC envoyé à l'adresse interne. |
| 2026-07-22 | J6b  | —   | 2251–2850 (offset=2250, limit=600)      | **1350 inchangé** | 600 max | **0/600**                                                                              | —            | ⛔ Non lancé | Pas de lancement manuel. Smoke 12:00 UTC envoyé à l'adresse interne. |

> Cumul confirmé au 22/07 à 16h38 Paris :
>
> - **Lots Channel Scope : 1 300 envoyés** (100 le 17/07 + 250 le 18/07 + 350 le 20/07 + 600 le 21/07).
> - **Newsletter Channel Scope, tests/smokes inclus : 1 312 messages vers 1 303 adresses uniques.**
> - **Total warm-up documenté incluant J1 interne : 1 342 messages** (30 internes + 1 300 lots + 12 tests/smokes).
> - **Aujourd'hui : 2 smokes, 0 envoi de masse.**
> - **Doublons dans les lots de la base : AUCUN** connu. L'adresse interne de smoke est répétée
>   volontairement plusieurs fois et n'est pas un doublon accidentel de campagne.
> - **Écart connu et accepté** : lignes 701–750 (50 adresses) ne seront jamais envoyées sur cette base (cf. décision 21/07 ci-dessus).

## Programmation jusqu'au mercredi 22/07 (fin de cette base)

On suit la rampe de [warm-up-strategie.md](./warm-up-strategie.md) (§ Plan J2-J7), **glissée d'un jour**
(décision 18/07) : dimanche = repos, car base B2B → ouvertures faibles le week-end = mauvais signaux
d'engagement pour Gmail. La rampe a été atténuée le 18/07 : progression de 400, puis 600, puis
900 + 600 maximum. La priorité est la réputation de `mail.attendee.fr`, pas l'épuisement de la base.

| Date       | Jour | Lignes CSV (offset→+limit)          | Jusqu'à ligne          | Volume | `--delay` conseillé | Durée estimée | Condition de lancement                                          |
| ---------- | ---- | ----------------------------------- | ---------------------- | ------ | ------------------- | ------------- | --------------------------------------------------------------- |
| 2026-07-19 | —    | **REPOS** (dimanche, base B2B)      | 350 (inchangé)         | 0      | —                   | —             | — (smoke test `rgharghar@choyou.fr` seulement si besoin)        |
| 2026-07-20 | J4   | ✅ **réel : 351–700** (limit réduit à 350, GO manquant → envoi manuel) | **700** (750 visé) | 400 | 20s | ~2h15 | signaux J3 propres (bounce faible, 0 plainte)          |
| 2026-07-21 | J5   | ✅ **réel : 751–1350**, 600/600 envoyés | **1350** | 600 | 15s | 2h31 réel | terminé à 13h23 Paris, exit 0                            |
| 2026-07-22 | J6a  | ⛔ **non envoyé** (GO absent)       | **1350 inchangé** | 0/900 | — | — | arrêt avant envoi au pré-contrôle de 09h Paris          |
| 2026-07-22 | J6b  | ⛔ **non envoyé**                   | **1350 inchangé** | 0/600 | — | — | aucun lancement manuel ; smoke de 14h seulement         |

> ⚠️ **J6a démarre bien à l'offset=1350 tel que planifié** : la fin de J5 (ligne 1350) n'est pas
> affectée par le raccourci de J4 (lignes 701–750 définitivement sautées, cf. décision 21/07 plus haut).
> Seul ce petit trou de 50 adresses est concerné, pas la suite de la rampe.

> **Réel mercredi soir à 16h38 : 1 342 messages de warm-up documentés** (30 internes + 1 300 lots
> Channel Scope + 12 tests/smokes). Sur les 5 220 adresses de la base, **3 920 n'ont reçu aucun lot** : les 50 lignes
> 701–750 volontairement sautées et les 3 870 lignes 1351–5220 non lancées. Jeudi 23/07, aucun nouvel
> envoi sans validation de la nouvelle base et des signaux consolidés.

### Historique de la programmation (timers posés le 18/07, désormais terminés)

Les lots avaient été programmés sur le VPS via `systemd-run` (timers one-shot, heures **UTC**) et passaient
par `scripts/ops/run-warmup-lot.sh` qui, pour chaque lot :

1. un timer séparé envoie le **smoke test à `rgharghar@choyou.fr` une heure avant** ;
2. vérifier réception, inbox/spam, contenu, liens et retours Mailgun ;
3. créer manuellement le fichier `GO` correspondant au lot ;
4. le timer du lot vérifie ce `GO` ; sans lui, il s'arrête avant tout envoi ;
5. (J6b uniquement) vérifier aussi ≥900 envois au journal du jour ;
6. lancer le lot et écrire `scripts/ops/warmup-run-<date>-offset<N>.log`.

Le timer du lot J6b avait été supprimé ; son smoke séparé est néanmoins parti à 12h00 UTC. J6b n'a
pas été lancé manuellement. Au 22/07 à 16h38, il ne reste **aucun timer warm-up actif**.

| Action       | Heure Paris | Heure VPS (UTC) | Paramètres / condition                         |
| ------------ | ----------- | --------------- | ---------------------------------------------- |
| Smoke J4     | lun. 08:00  | lun. 06:00      | interne uniquement                             |
| Lot J4       | lun. 09:00  | lun. 07:00      | offset=350, limit=400, 20s + `GO`              |
| Smoke J5     | mar. 08:00  | mar. 06:00      | interne uniquement                             |
| Lot J5       | mar. 09:00  | mar. 07:00      | offset=750, limit=600, 15s + `GO`              |
| Smoke J6a    | mer. 08:00  | mer. 06:00      | interne uniquement                             |
| Lot J6a      | mer. 09:00  | mer. 07:00      | offset=1350, limit=900, 10s + `GO`             |
| Smoke J6b    | mer. 14:00  | mer. 12:00      | interne uniquement                             |
| Lot J6b      | mer. 15:00 cible | lancement manuel | offset=2250, limit=600, 10s + `GO`          |

Le VPS est configuré en `Etc/UTC`. En juillet, Paris est en `CEST` (`UTC+2`) : **07:00 UTC = 09:00
Paris**. Cette conversion a été vérifiée directement sur le serveur le 18/07 avec
`TZ=Europe/Paris date`. Les heures de référence métier dans ce document sont toujours celles de Paris.

### Validation après réception du smoke test

Avant de créer le fichier `GO`, contrôler :

- mail reçu dans la boîte principale, pas en spam ;
- expéditeur `ChannelScope <no-reply@mail.attendee.fr>`, objet, preheader et rendu corrects ;
- images et liens fonctionnels, notamment le lien de désinscription ;
- événements Mailgun `accepted` puis `delivered` ; `opened` est attendu après ouverture ;
- aucun nouvel événement `failed`, `complained` ou blocage provider anormal.

Fichiers `GO` des lots automatiques :

```bash
# Après contrôle du smoke lundi
ssh ems-vps 'printf "GO 2026-07-20 350\n" > /tmp/warmup-go-2026-07-20-350'

# Après contrôle du smoke mardi
ssh ems-vps 'printf "GO 2026-07-21 750\n" > /tmp/warmup-go-2026-07-21-750'

# Après contrôle du smoke mercredi matin
ssh ems-vps 'printf "GO 2026-07-22 1350\n" > /tmp/warmup-go-2026-07-22-1350'
```

Après décision humaine `GO` mercredi, autoriser puis lancer J6b :

```bash
ssh ems-vps 'printf "GO 2026-07-22 2250\n" > /tmp/warmup-j6b-go-2026-07-22'
ssh ems-vps 'nohup /opt/ems-attendee/backend/scripts/ops/run-warmup-lot.sh \
  2250 600 10 900 /tmp/warmup-j6b-go-2026-07-22 1 >/tmp/warmup-j6b-launch.log 2>&1 &'
```

Sans fichier au contenu exact `GO 2026-07-22 2250`, le runner s'arrête **avant le smoke test** et
avant tout envoi du lot.

### Calendrier des contrôles (heure de Paris)

| Date / heure                         | Action                                                                |
| ------------------------------------ | --------------------------------------------------------------------- |
| Dim. 19/07, 10h-12h                 | bilan J1-J3 et décision sur J4                                        |
| Lun. 20/07, 08h30 / 11h / 17h-18h  | pré-GO J4 / contrôle rapide / bilan consolidé                         |
| Mar. 21/07, 08h30 / 11h-12h / 17h-18h | pré-GO J5 / contrôle rapide / bilan consolidé                      |
| Mer. 22/07, avant 08h30             | pré-GO J6a                                                           |
| Mer. 22/07, 11h-14h                 | contrôle J6a et décision humaine `GO / HOLD / STOP` pour J6b          |

À chaque contrôle : compter `accepted`, `delivered`, `temporary/permanent failure`, `complained` et
`unsubscribed`, puis vérifier leur concentration par provider. Appliquer le feu défini dans
[warm-up-strategie.md](./warm-up-strategie.md) : vert = continuer, orange = maintenir/réduire,
rouge = arrêter.

Contrôle / annulation (depuis le Mac, alias SSH `ems-vps`) :

```bash
ssh ems-vps 'systemctl list-timers warmup-\* --no-pager'          # voir les timers
ssh ems-vps 'sudo systemctl stop warmup-j4.timer'                  # annuler UN lot
ssh ems-vps 'sudo systemctl stop "warmup-*.timer"'                 # tout annuler
ssh ems-vps 'tail -50 /opt/ems-attendee/backend/scripts/ops/warmup-run-*.log'  # suivre un run
```

## Comment mettre à jour ce tableau après un lot

1. Compter les succès : `grep -c "✅" <log_du_run>` (ou `wc -l` sur le journal CSV du jour moins 1 pour
   l'en-tête, si un seul lot dans la journée).
2. Ajouter/compléter la ligne correspondante ci-dessus.
3. Noter tout signal négatif (bounce, complaint, deferred anormal) dans la colonne "Signaux observés" —
   vérifiable via le dashboard Mailgun ou un appel à l'API events (`/v3/<domain>/events?recipient=...`).
4. Si un lot échoue en cours de route (process tué, erreur), noter la ligne exacte où ça s'est arrêté
   pour que le prochain `--offset` reparte au bon endroit (ne jamais se fier uniquement au `--limit`
   prévu si le run n'est pas allé au bout).
