# Suivi des envois warm-up — base Channel Scope

> **À QUOI SERT CE FICHIER :** source de vérité unique pour savoir **exactement quelles lignes de la
> base ont été envoyées quel jour**. Avant ce fichier, on ne pouvait retrouver l'historique qu'en
> fouillant les journaux CSV sur le VPS — ce qui a créé une confusion (J2 vs J3) le 18/07.
> **Toujours mettre à jour ce tableau après chaque lot envoyé.**
>
> Stratégie / paliers cibles → [warm-up-strategie.md](./warm-up-strategie.md)
> Détails techniques / risques → [c3-warmup-delivrabilite.md](./c3-warmup-delivrabilite.md)

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
  lot relancé à **08:52 UTC** (smoke du matin 06:00 UTC déjà OK, non répété).

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

## Tableau de suivi (source de vérité)

| Date       | Jour | Lot | Lignes CSV (offset→+limit)              | Jusqu'à ligne | Ciblé | Envoyés OK                                                                                | Échecs       | Statut      | Signaux observés                                                                     |
| ---------- | ---- | --- | --------------------------------------- | ------------- | ----- | ----------------------------------------------------------------------------------------- | ------------ | ----------- | ------------------------------------------------------------------------------------ |
| 2026-07-16 | J1   | —   | Liste interne (hors base Channel Scope) | —             | 30    | 30                                                                                        | 0            | ✅ Terminé  | Liste interne uniquement (warm-up initial)                                           |
| 2026-07-17 | J2   | 1   | 1–100 (offset=0, limit=100)             | **100**       | 100   | 100                                                                                       | 0            | ✅ Terminé  | Propre — pas d'alerte documentée                                                     |
| 2026-07-18 | J3   | 2   | 101–200 (offset=100, limit=100)         | **200**       | 100   | 100/100 (journal `warmup-wave2-sent-2026-07-18.csv`, +1 smoke test `rgharghar@choyou.fr`) | 0            | ✅ Terminé  | Smoke test 18/07 : accepted→delivered→opened en ~5s, propre. Aucun doublon (vérifié) |
| 2026-07-18 | J3   | 3   | 201–350 (offset=200, limit=150)         | **350**       | 150   | 150/150 (journal `warmup-wave2-sent-2026-07-18.csv`)                                       | 0            | ✅ Terminé  | Rythme ~1 email/30s. Aucun doublon vérifié                                            |
| 2026-07-20 | J4   | 4   | 351–700 (offset=350, **limit=350 réel** — 400 prévu) | **700** (750 prévu) | 400 prévu / 350 réel | 350/350 (journal `warmup-wave2-sent-2026-07-20.csv`, +1 smoke `rgharghar@choyou.fr` 06:00 UTC) | 8 permanents + 5 temporaires (13, cumul J4+smoke J5) | ✅ Terminé (hors mécanisme `GO`) | Timer auto 07:00 UTC **annulé** (fichier `GO` absent). Lot relancé **manuellement** 07:53→10:47 UTC avec `limit` réduit à 350 → écart lignes 701–750 (50 adresses) jamais envoyées. 0 plainte spam, ~55 % d'ouverture |
| 2026-07-21 | J5   | 5   | 751–1350 (offset=750, limit=600)        | **1350**      | 600   | 🟡 en cours (lot lancé 08:52 UTC, GO créé a posteriori)                                     | 0 à ce stade | 🟡 En cours | Timer auto 07:00 UTC **annulé** (fichier `GO` absent, deadline 09h Paris ratée). Smoke 06:00 UTC OK. `GO 2026-07-21 750` créé manuellement, lot relancé avec `skip_smoke=1`. Écart J4 (701–750) **non rattrapé** (décision : option B) |

> Cumul confirmé à ce stade (21/07 ~08:55 UTC) :
>
> - **Channel Scope : 700 envoyés** (100 le 17/07 + 250 le 18/07 + 350 le 20/07, hors smoke tests) + J5 en cours (751–1350, 600 visés).
> - **Total warm-up (interne + Channel Scope) : 730 envoyés** (30 + 700) + J5 en cours.
> - **Doublons : AUCUN** connu à ce stade (dédup interne au journal du jour).
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
| 2026-07-21 | J5   | 🟡 **en cours : 751–1350** (offset=750, limit=600, GO créé a posteriori) | **1350** | 600 | 15s | ~2h30 | deferred faible, inbox OK Gmail/Outlook                |
| 2026-07-22 | J6a  | 1351–2250 (offset=1350, limit=900)  | **2250** | 900 | 10s | ~2h30 | queue/webhooks stables, pas de blocage                 |
| 2026-07-22 | J6b  | 2251–2850 (offset=2250, limit=600)  | **2850** | 600 max | 10s | ~1h40 | **GO humain obligatoire** après analyse Mailgun de J6a |

> ⚠️ **J6a démarre bien à l'offset=1350 tel que planifié** : la fin de J5 (ligne 1350) n'est pas
> affectée par le raccourci de J4 (lignes 701–750 définitivement sautées, cf. décision 21/07 plus haut).
> Seul ce petit trou de 50 adresses est concerné, pas la suite de la rampe.

> **Total maximal mercredi soir : 2 880 emails de warm-up** (30 internes + 2850 Channel Scope), hors
> smoke tests. Il restera 2370 adresses Channel Scope non envoyées. Elles ne sont pas un objectif à
> épuiser : jeudi 23/07, aucun nouvel envoi sans validation de la base et des signaux consolidés.

### ✅ Programmation EFFECTIVE (timers systemd posés le 18/07)

Les 4 lots sont programmés sur le VPS via `systemd-run` (timers one-shot, heures **UTC**) et passent
par `scripts/ops/run-warmup-lot.sh` qui, pour chaque lot :

1. un timer séparé envoie le **smoke test à `rgharghar@choyou.fr` une heure avant** ;
2. vérifier réception, inbox/spam, contenu, liens et retours Mailgun ;
3. créer manuellement le fichier `GO` correspondant au lot ;
4. le timer du lot vérifie ce `GO` ; sans lui, il s'arrête avant tout envoi ;
5. (J6b uniquement) vérifier aussi ≥900 envois au journal du jour ;
6. lancer le lot et écrire `scripts/ops/warmup-run-<date>-offset<N>.log`.

Le timer J6b est supprimé. Après validation des signaux J6a, créer le fichier d'autorisation puis
lancer manuellement le runner. Sans validation avant 15h heure de Paris, J6b est reporté.

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
