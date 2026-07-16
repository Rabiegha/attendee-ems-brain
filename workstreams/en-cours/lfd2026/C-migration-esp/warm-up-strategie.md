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

| Jour | Volume cible | Condition |
| ---- | ------------ | --------- |
| J1   | 20-50        | setup + premiers signaux |
| J2   | ~100         | pas de plainte, bounce faible |
| J3   | ~250         | inbox OK Gmail/Outlook |
| J4   | ~500         | deferred faible |
| J5   | ~1 000       | queue et webhooks stables |
| J6   | ~2 000       | pas de blocage provider |
| J7+  | progressif   | montée vers besoin event |

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

### Plan indicatif semaine 2

Hypothèse : J1-J7 propres, aucune plainte spam, bounce faible, pas de blocage Gmail/Outlook/Yahoo.

| Jour | Volume cible | Débit conseillé | Condition |
| ---- | ------------ | --------------- | --------- |
| S2-J1 | ~1 000-1 500 | 1 email/s | reprendre sans saut brutal après J7 |
| S2-J2 | ~1 500 | 1 email/s | delivered OK, pas de deferred anormal |
| S2-J3 | ~2 000 | 1-2 emails/s | Gmail/Outlook toujours OK |
| S2-J4 | ~2 000 | 1-2 emails/s | maintenir si signaux moyens |
| S2-J5 | ~2 500 | 2 emails/s max | seulement si signaux verts |
| S2-J6 | pause ou petit lot ~500 | 1 email/s | respiration + analyse provider |
| S2-J7 | ~2 500-3 000 | 2 emails/s max | premier palier proche du besoin event |

### Plan indicatif semaine 3

Objectif : valider que `mail.attendee.fr` et le chemin Attendee/Mailgun tiennent des volumes proches
du besoin LFD, puis tester la fenêtre opérationnelle.

| Jour | Volume cible | Débit conseillé | Condition |
| ---- | ------------ | --------------- | --------- |
| S3-J1 | ~2 000 | 1-2 emails/s | reprise prudente |
| S3-J2 | ~3 000 | 2 emails/s max | test volume cible |
| S3-J3 | ~3 000 | 2 emails/s max | confirmer stabilité providers |
| S3-J4 | pause ou petit lot ~500-1 000 | 1 email/s | analyse + nettoyage base |
| S3-J5 | ~3 000 | 2-3 emails/s max | test de répétabilité |
| S3-J6 | test fenêtre : ~3 000 en 30-60 min | 1-2 emails/s selon fenêtre | vérifier queue/webhooks/backlog |
| S3-J7 | ajustement | selon signaux | réduire, maintenir ou préparer S4 |

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
