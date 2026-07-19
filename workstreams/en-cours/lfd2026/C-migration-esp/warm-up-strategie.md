# Stratégie warm-up Mailgun — `mail.attendee.fr`

> Date : 15/07/2026
> Domaine : `mail.attendee.fr`
> ESP : Mailgun EU, IP partagée
> Source warm-up disponible : base newsletter du magasin Channel Scope

## Principe

Le but n'est pas seulement d'envoyer du volume. Le but est de créer de bons signaux :

- peu ou pas de hard bounces ;
- pas de plaintes spam ;
- ouvertures/clics naturels ;
- contenu attendu par les destinataires ;
- montée progressive par fournisseur destinataire.

## Contrainte cible LFD : pics autour de 3 000 emails

⚠️ Point à ne pas oublier : le warm-up ne vise pas seulement quelques centaines d'emails. Pour LFD,
on anticipe des **pics autour de 3 000 emails** à absorber proprement (ex. vague d'inscriptions,
relances, billets/QR, notifications).

Conséquence :

- le domaine `mail.attendee.fr` doit progressivement être exposé à des volumes journaliers proches
  ou supérieurs à 3 000 emails avant l'event ;
- le système Attendee doit tenir le chemin `queue -> Mailgun -> webhooks` pendant ces volumes ;
- la montée J1-J7 est un démarrage, pas la cible finale ;
- après J7, continuer sur 2-4 semaines avec des paliers vers 3 000/jour puis des tests ponctuels
  proches du pic attendu.

Important : **volume** et **débit** sont deux réglages différents.

- Volume = nombre d'emails envoyés dans la journée.
- Débit = vitesse d'envoi (`email:rate_limit_per_second`).

On peut envoyer 3 000 emails à `1 email/s` en environ 50 minutes. Il n'est donc pas nécessaire de
monter brutalement le débit si la fenêtre d'envoi le permet. La priorité est d'abord la réputation
provider, puis la stabilité queue/webhooks, puis seulement le débit.

## Base Channel Scope

Bonne nouvelle : la base newsletter Channel Scope peut servir au warm-up si elle est utilisée
proprement.

Conditions avant envoi :

- les destinataires doivent avoir accepté de recevoir des emails de Channel Scope ou d'un contexte
  explicitement relié ;
- l'email doit être cohérent avec cette relation, pas un email Attendee/LFD sorti de nulle part ;
- inclure un lien ou une procédure de désinscription claire ;
- éviter les adresses anciennes, inconnues, génériques ou suspectes ;
- commencer par les contacts les plus engagés si cette information existe.

⚠️ Point de vigilance : warm-up de `mail.attendee.fr` avec une base Channel Scope peut être utile
techniquement, mais il faut éviter de provoquer des plaintes parce que le destinataire ne reconnaît
pas l'expéditeur. Le contenu doit donc expliquer clairement le contexte Channel Scope et rester attendu.

## Qu'est-ce qui fait le warm-up ?

Le warm-up n'est pas un bouton Mailgun. Ce qui chauffe le domaine, ce sont des **vrais envois**
vers des destinataires réels, avec de bons signaux : livraison, peu de bounces, peu de plaintes,
ouvertures/clics naturels.

Le mécanisme recommandé côté Attendee :

```txt
CSV/liste Channel Scope nettoyée
-> script/commande interne de warm-up
-> queue BullMQ email-send
-> EmailService Attendee
-> Mailgun API EU
-> destinataires réels
-> webhooks Mailgun
-> email_events + stats par domaine destinataire
```

Le script ne "chauffe" pas magiquement le domaine : il sert seulement à **déclencher proprement**
les envois, par lots, en respectant le throttle et l'idempotence. La réputation vient ensuite des
réactions mailbox providers et destinataires.

À éviter :

- boucle `curl` directe vers Mailgun pour toute la base ;
- envoi depuis un outil externe qui contourne la queue Attendee ;
- gros import sans idempotence ;
- envoi sans pouvoir pauser / reprendre / tracer.

Option acceptable pour les 5 premiers tests : envoi manuel Mailgun ou commande ponctuelle.
Pour J1 réel et la suite, utiliser un **petit script contrôlé** qui passe par le chemin applicatif.

## Plan J1

Objectif : 20 à 50 emails maximum.

Débit :

```txt
email:rate_limit_per_second = 1
email:paused = false
```

Séquence :

1. Lot 1 : 5 emails internes / adresses sûres.
2. Attendre 10-15 min, vérifier inbox/spam et Mailgun events.
3. Lot 2 : 10-15 emails Channel Scope très sûrs.
4. Attendre 30 min, vérifier Gmail/Outlook/domaines pro séparément.
5. Lot 3 : compléter jusqu'à 25-40 emails si tout est propre.

Stopper J1 si :

- `spam complaints` > 0 ;
- plusieurs `permanent failure` ;
- beaucoup de `temporary failure` chez un provider ;
- placement spam évident chez Gmail/Outlook.

## Plan J2-J7 indicatif

À ajuster selon les métriques réelles. Ce tableau est une **rampe de démarrage**, pas l'objectif final :
le besoin LFD impose ensuite de monter vers des journées autour de 3 000 emails si les signaux restent
propres.

> 📌 **Exécution réelle (mise à jour 18/07)** : J1 = 30 internes (16/07), J2 = 100 (17/07), J3 = 250 (18/07).
> La suite est **glissée d'un jour** (dimanche 19/07 = repos, base B2B → ouvertures faibles le week-end) :
> Rampe atténuée validée le 18/07 : J4 = 400 (lun 20/07), J5 = 600 (mar 21/07),
> J6 = 900 + 600 maximum (mer 22/07, second lot sous `GO` humain explicite).
> Le palier "J7 progressif" est reporté sur la **nouvelle base à partir du jeudi 23/07**.
> Programmation et état au jour le jour → [suivi-envois-channelscope.md](./suivi-envois-channelscope.md).

| Jour | Volume cible | Condition                     |
| ---- | ------------ | ----------------------------- |
| J1   | 20-50        | setup + premiers signaux      |
| J2   | ~100         | pas de plainte, bounce faible |
| J3   | ~250         | inbox OK Gmail/Outlook        |
| J4   | ~400         | deferred faible               |
| J5   | ~600         | queue et webhooks stables     |
| J6   | ~900 + 600 max | pas de blocage provider + GO humain avant le second lot |
| J7+  | progressif   | montée vers besoin event      |

## Après J7 — montée vers le besoin event

Si aucun signal rouge n'apparaît, continuer la montée sur 2-4 semaines :

- viser progressivement 1 500 -> 2 000 -> 3 000 emails/jour ;
- garder `1 email/s` au début, puis passer à `2 email/s` uniquement si Gmail/Outlook/Yahoo restent OK ;
- ne pas augmenter volume et débit le même jour si les signaux sont ambigus ;
- faire au moins un test contrôlé proche de 3 000 emails avant l'event ;
- si 3 000 emails/jour passent proprement, tester ensuite la fenêtre opérationnelle réellement voulue
  (ex. 3 000 emails en ~30-60 min selon throttle).

Règle de montée :

- signaux verts : +30-50 % de volume au prochain palier ;
- signaux moyens : maintenir le même volume ;
- `temporary failure` élevé, spam placement ou complaint : pause/réduction et analyse par provider ;
- `spam complaints` > 0 : stop immédiat du lot et revue de la base/contenu.

### Fenêtres de contrôle obligatoires avant chaque palier

L'absence de doublons protège les destinataires contre la répétition, mais ne remplace pas le suivi
de réputation : les bounces, plaintes et blocages sont attachés au domaine d'envoi, à l'IP et aux
mailbox providers. Certains signaux arrivent avec plusieurs heures de retard.

Pour chaque lot, effectuer trois contrôles :

1. **une heure avant le départ** : smoke test interne, vérification visuelle, événements Mailgun et
   décision humaine `GO / HOLD / STOP` ;
2. **1 à 2 heures après le départ** : détection rapide d'un blocage ou de `temporary failure` ;
3. **en fin de journée ou le lendemain matin** : bilan consolidé incluant les retours tardifs.

Décision :

- **VERT — GO** : 0 plainte, bounces permanents faibles, temporaires faibles et dispersés, aucune
  concentration anormale chez Gmail/Outlook/Yahoo ou un domaine professionnel ;
- **ORANGE — HOLD/RÉDUCTION** : hausse des temporaires, ralentissement de livraison, blocage concentré
  chez un provider ou signaux encore incomplets ; maintenir ou réduire le volume, sans accélérer ;
- **ROUGE — STOP** : plainte spam, blocage provider, hausse nette des bounces permanents ou placement
  spam évident ; annuler le lot suivant et analyser la base, le contenu et le provider concerné.

Un smoke test valide seulement le chemin technique. Il ne vaut jamais validation de réputation et
ne peut pas, à lui seul, autoriser le palier suivant.

### Comment donner le `GO`

Après avoir reçu et contrôlé le smoke test, le `GO` doit être donné avant 09h heure de Paris. Deux
méthodes sont possibles :

1. écrire à Codex `GO J4`, `GO J5` ou `GO J6a` ; Codex vérifie d'abord les événements Mailgun puis
   crée l'autorisation sur le VPS ;
2. créer directement le fichier d'autorisation depuis un terminal :

```bash
# Lundi — J4, lot de 400
ssh ems-vps 'printf "GO 2026-07-20 350\n" > /tmp/warmup-go-2026-07-20-350'

# Mardi — J5, lot de 600
ssh ems-vps 'printf "GO 2026-07-21 750\n" > /tmp/warmup-go-2026-07-21-750'

# Mercredi matin — J6a, lot de 900
ssh ems-vps 'printf "GO 2026-07-22 1350\n" > /tmp/warmup-go-2026-07-22-1350'
```

Le contenu doit correspondre exactement à la date et à l'offset attendus. Sans fichier valide au
moment du timer de 09h, le lot est annulé avant tout envoi. Ajouter le fichier après 09h ne relance
pas automatiquement le lot : il faudra alors décider d'un lancement manuel ou d'un report.

### Plan indicatif semaine 2

Hypothèse : J1-J7 propres, aucune plainte spam, bounce faible, pas de blocage Gmail/Outlook/Yahoo.

| Jour  | Volume cible            | Débit conseillé | Condition                             |
| ----- | ----------------------- | --------------- | ------------------------------------- |
| S2-J1 | ~1 000-1 500            | 1 email/s       | reprendre sans saut brutal après J7   |
| S2-J2 | ~1 500                  | 1 email/s       | delivered OK, pas de deferred anormal |
| S2-J3 | ~2 000                  | 1-2 emails/s    | Gmail/Outlook toujours OK             |
| S2-J4 | ~2 000                  | 1-2 emails/s    | maintenir si signaux moyens           |
| S2-J5 | ~2 500                  | 2 emails/s max  | seulement si signaux verts            |
| S2-J6 | pause ou petit lot ~500 | 1 email/s       | respiration + analyse provider        |
| S2-J7 | ~2 500-3 000            | 2 emails/s max  | premier palier proche du besoin event |

### Plan indicatif semaine 3

Objectif : valider que `mail.attendee.fr` et le chemin Attendee/Mailgun tiennent des volumes proches
du besoin LFD, puis tester la fenêtre opérationnelle.

| Jour  | Volume cible                       | Débit conseillé            | Condition                         |
| ----- | ---------------------------------- | -------------------------- | --------------------------------- |
| S3-J1 | ~2 000                             | 1-2 emails/s               | reprise prudente                  |
| S3-J2 | ~3 000                             | 2 emails/s max             | test volume cible                 |
| S3-J3 | ~3 000                             | 2 emails/s max             | confirmer stabilité providers     |
| S3-J4 | pause ou petit lot ~500-1 000      | 1 email/s                  | analyse + nettoyage base          |
| S3-J5 | ~3 000                             | 2-3 emails/s max           | test de répétabilité              |
| S3-J6 | test fenêtre : ~3 000 en 30-60 min | 1-2 emails/s selon fenêtre | vérifier queue/webhooks/backlog   |
| S3-J7 | ajustement                         | selon signaux              | réduire, maintenir ou préparer S4 |

Ces volumes sont des **plafonds conditionnels**. Si la base disponible n'est pas assez propre, il vaut
mieux rester sous ces volumes que créer de mauvais signaux de réputation.

## Contenu recommandé

Utiliser un vrai contenu, pas un email technique vide.

Exemples acceptables :

- newsletter courte Channel Scope ;
- annonce ou information utile attendue par les inscrits ;
- email sobre avec branding clair, adresse de contact, désinscription.

À éviter :

- sujet trompeur ;
- contenu LFD envoyé à une base qui ne connaît pas LFD ;
- pièces jointes lourdes au début ;
- répétition du même email à la même personne ;
- gros volume sur des adresses non engagées.

## Monitoring

À suivre après chaque lot :

- accepted ;
- delivered ;
- temporary failure ;
- permanent failure ;
- spam complaints ;
- open/click si tracking activé ;
- Gmail vs Outlook vs Yahoo vs domaines pro ;
- délai inscription/enqueue -> accepted -> delivered.
- capacité à absorber une vague proche de 3 000 emails sans backlog durable, sans explosion webhook
  et sans blocage provider.

Pour le calendrier opérationnel du 19 au 22 juillet 2026 :

| Moment (heure de Paris)               | Contrôle / décision                                                      |
| ------------------------------------- | ------------------------------------------------------------------------ |
| Dim. 19/07, 10h-12h                  | bilan consolidé J1-J3 ; autoriser ou non les 400 du lundi                |
| Lun. 20/07, 08h30 / 11h / 17h-18h   | pré-GO J4, contrôle rapide, puis bilan consolidé                         |
| Mar. 21/07, 08h30 / 11h-12h / 17h-18h | pré-GO J5, contrôle rapide, puis bilan consolidé                       |
| Mer. 22/07, avant 08h30              | autoriser ou non les 900 de J6a                                          |
| Mer. 22/07, 11h-14h                  | analyser J6a et décider manuellement `GO / HOLD / STOP` pour J6b         |

**J6b ne doit jamais partir sur le seul critère « 900 lignes journalisées ».** Sa validation doit
inclure les événements Mailgun (`delivered`, `failed`, `complained`, `unsubscribed`, `deferred`) et
la répartition par provider. En l'absence de validation humaine explicite avant 15h heure de Paris,
le lot est reporté.

## Faut-il warmer deux domaines ?

Réponse courte : **non, pas pour le jour de l'event sauf besoin très clair**.

Pour LFD 2026, garder un domaine principal warmé est préférable :

```txt
mail.attendee.fr
```

Raisons :

- la réputation se construit par domaine/sous-domaine ;
- diviser le volume entre plusieurs domaines fragmente les signaux ;
- chaque domaine doit avoir SPF/DKIM/DMARC, tracking, webhooks, monitoring ;
- un domaine secondaire non suffisamment warmé n'est pas un vrai secours ;
- le jour J, la simplicité opérationnelle vaut cher.

Un deuxième domaine peut avoir du sens seulement si :

- il sépare clairement deux flux avec audiences différentes ;
- il est warmé plusieurs semaines avant ;
- il a ses propres DNS, webhooks, monitoring et runbook ;
- on accepte la complexité supplémentaire.

Décision recommandée :

- **Primary event** : `mail.attendee.fr`.
- **Pas de deuxième domaine pour LFD 2026** sauf incident ou contrainte nouvelle.
- Reconsidérer post-event si Attendee veut séparer transactionnel, marketing, clients white-label.
