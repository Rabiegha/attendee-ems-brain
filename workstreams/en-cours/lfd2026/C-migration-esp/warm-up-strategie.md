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

À ajuster selon les métriques réelles.

| Jour | Volume cible | Condition |
| ---- | ------------ | --------- |
| J1   | 20-50        | setup + premiers signaux |
| J2   | ~100         | pas de plainte, bounce faible |
| J3   | ~250         | inbox OK Gmail/Outlook |
| J4   | ~500         | deferred faible |
| J5   | ~1 000       | queue et webhooks stables |
| J6   | ~2 000       | pas de blocage provider |
| J7+  | progressif   | montée vers besoin event |

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
