# N-ANTI — Protection anti-bot et anti-abus LFD

> **Parent :** [chantier N — architecture event-ready](./README.md)
> **Horizon :** avant LFD 2026
> **Owner proposé :** Rabie
> **Priorité :** P1 event, avant le test k6 final
> **Estimation initiale :** 2,5–4,5 j-dev

## 1. Pourquoi ce sous-chantier existe

Le CDC demande une protection contre les bots, le scraping massif et les inscriptions automatisées,
par CAPTCHA ou mécanisme équivalent. Le besoin n'est pas de construire une plateforme antifraude :
il faut empêcher un script de remplir les jauges tout en laissant passer un pic légitime de plusieurs
milliers d'utilisateurs, dont certains peuvent partager l'IP d'un réseau ministériel.

## 2. État actuel vérifié

- les routes publiques Attendee event et session ont un `@Throttle` de 100 requêtes/minute par tracker ;
- ce plafond a été volontairement laissé haut pour les NAT d'entreprise ;
- aucun challenge anti-bot n'est présent dans le formulaire BIL ;
- aucune validation Turnstile/CAPTCHA n'est faite côté serveur ;
- l'anti-doublon métier existe, mais ne bloque pas un bot utilisant des emails différents ;
- aucune vraie idempotency key de soumission n'est documentée sur le flux BIL ;
- le navigateur appelle `back-office-event/register.php`, qui appelle ensuite Attendee en PHP.

### Risque proxy à résoudre avant k6

`back-office-event/admin/attendee.php` ne transmet pas aujourd'hui l'IP du visiteur à Attendee.
Le throttler Attendee risque donc de voir toutes les inscriptions comme provenant du serveur PHP et
d'appliquer le plafond de 100/min à l'ensemble de LFD. Il ne faut pas transmettre aveuglément un
`X-Forwarded-For` fourni par le navigateur : il serait falsifiable.

À titre d'ordre de grandeur, 100/min sur une IP agrégée représente seulement ~1,67 requête/s pour toute
la billetterie. Les anciens tests k6 directs vers Attendee ne prouvent pas que le parcours PHP réel
échappe à ce plafond. Ce point devient un prérequis du test bout en bout BIL.

Le lot N-ANTI doit choisir et tester une seule cible :

1. **Cible recommandée après C2.1 : appel navigateur → Attendee directement**, avec CORS limité au
   domaine BIL, Turnstile vérifié dans Attendee et IP reçue via le proxy de confiance ; ou
2. **Proxy PHP conservé :** rate limit à l'entrée de `register.php` avec l'IP réellement observée par
   le serveur BIL, et transmission éventuelle d'un contexte signé/accepté uniquement depuis le serveur
   BIL. Ne jamais faire confiance à un header client arbitraire.

Le choix doit être figé avant le test k6 final. Le test ciblé L9.b peut continuer à appeler Attendee
directement et ne doit pas être bloqué par Turnstile grâce aux clés/modes de test dédiés.

## 3. Architecture MVP retenue

```text
Navigateur BIL
  -> challenge Turnstile transparent/managed
  -> request_id d'inscription
  -> endpoint public autoritatif
       1. rate limits progressifs
       2. validation Turnstile côté serveur
       3. idempotence de la soumission
       4. règles métier et transaction L9.b
       5. réponse immédiate
  -> effets secondaires BullMQ après commit
```

Turnstile n'est qu'une couche. Il ne remplace ni le rate limiting, ni l'idempotence, ni les règles
métier de capacité/anti-doublon.

## 4. Protections par couche

### A. Turnstile

- widget en mode transparent/managed sur la soumission du formulaire ;
- token envoyé avec l'inscription ;
- appel `Siteverify` uniquement côté serveur avec secret en variable d'environnement ;
- vérifier `success`, le hostname et une action dédiée telle que `lfd_register_session` ;
- timeout court et erreurs différenciées : token invalide, expiré/réutilisé, fournisseur indisponible ;
- ne jamais exposer la secret key au navigateur ;
- clés différentes staging/prod et clés officielles de test pour E2E/k6.

Selon la documentation Cloudflare consultée le 19/07/2026, la validation serveur est obligatoire,
les tokens sont à usage unique et expirent après cinq minutes. Référence :
[Cloudflare — server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/).

### B. Rate limiting progressif

Ne pas utiliser une limite basse par IP comme règle principale.

| Signal | Rôle | Décision initiale |
| --- | --- | --- |
| IP réellement observée | bloquer un flood évident | plafond haut, compatible NAT ; valeur après k6 |
| email normalisé + session | limiter répétitions ciblées | petite fenêtre glissante, sans révéler si l'email existe |
| `request_id` | reconnaître le retry légitime | même résultat, pas de seconde transaction |
| token Turnstile | preuve anti-automatisation | obligatoire en mode normal/attack |
| session ciblée | détecter une jauge attaquée | métrique/alerte, pas blocage métier seul |

Les compteurs partagés doivent vivre dans Redis si plusieurs processus les consomment. Les clés ne
doivent pas stocker l'email en clair : utiliser un HMAC/hash avec secret et TTL court.

### C. Idempotence métier

- le navigateur génère un UUID `request_id` avant le premier envoi ;
- le même UUID est réutilisé après timeout/retry ;
- Attendee retourne le résultat original si la même demande a déjà été commitée ;
- une même clé avec un payload différent est refusée ;
- TTL suffisamment long pour couvrir les retries et le pic ;
- ne pas confondre cette clé avec l'`idempotency_key` optionnelle de Siteverify, qui ne sécurise que
  le retry de validation Turnstile.

### D. Règles métier

Restent synchrones dans Attendee : unicité, capacité, waitlist, ouverture/fermeture et règles LFD
validées avec le client. La règle « deux activités `confirmed`/jour » est portée par H-D8. N-ANTI porte
uniquement l'interprétation appareil/IP comme signal anti-abus. Le `+1` reste à trancher au call client
du 21/07 puis sera réparti entre H/BIL et B/C2.1/D.

## 5. Modes opérationnels

| Mode | Turnstile | Rate limits | Usage |
| --- | --- | --- | --- |
| `off` | non exigé | garde-fous minimaux | développement/local uniquement |
| `normal` | actif | compatibles trafic courant | fonctionnement habituel |
| `event` | actif | plafond IP relevé, autres signaux conservés | ouverture LFD |
| `attack` | actif/renforcé | seuils plus stricts | incident confirmé |

Variables proposées :

```text
ANTI_BOT_ENABLED
ANTI_BOT_MODE
TURNSTILE_SITE_KEY
TURNSTILE_SECRET_KEY
TURNSTILE_EXPECTED_HOSTNAME
TURNSTILE_EXPECTED_ACTION
TURNSTILE_TIMEOUT_MS
```

Les seuils rate limit doivent rester dans une configuration validée/versionnée. Ne pas rendre des
valeurs sensibles modifiables librement depuis le navigateur ou le back-office client.

## 6. Mode dégradé et UX

### Token refusé ou expiré

- ne pas consommer de place ;
- afficher une erreur compréhensible ;
- régénérer le challenge et permettre une nouvelle soumission avec le même `request_id` tant qu'aucun
  commit métier n'a eu lieu.

### Fournisseur Turnstile indisponible

Décision recommandée pour la disponibilité LFD : **fail-open contrôlé**, uniquement pour une erreur
réseau/interne du fournisseur, jamais pour un token explicitement invalide.

- journaliser l'entrée en mode dégradé ;
- conserver rate limits, idempotence et règles métier ;
- déclencher une alerte ;
- augmenter temporairement la surveillance ;
- permettre un passage opérateur vers `event`/`attack` selon le runbook.

Le choix fail-open/fail-closed doit être validé avant production et testé. En cas de doute contractuel,
le runbook doit faire primer la disponibilité sans désactiver toutes les autres protections.

### UX de soumission

- désactiver le bouton après clic ;
- afficher « Vérification puis inscription en cours… » ;
- ne pas exposer la raison interne d'un refus anti-bot ;
- `429` : indiquer une attente et un retry borné ;
- timeout : vérifier le statut via `request_id` avant d'autoriser une nouvelle tentative.

## 7. Observabilité et alertes

Métriques minimales :

- validations Turnstile succès/échec/timeout ;
- erreurs par code agrégé, sans token ni donnée personnelle ;
- requêtes acceptées/refusées par couche ;
- `429` par endpoint et par fenêtre ;
- retries/idempotency hits/conflits de payload ;
- débit d'inscriptions valides par session ;
- latence ajoutée par Siteverify et p95/p99 total ;
- activation d'un mode dégradé et durée.

Alertes : hausse anormale des refus, indisponibilité Siteverify, `429` touchant une part significative
des utilisateurs, écart important entre tentatives et inscriptions valides, ou changement de mode.

## 8. Tests obligatoires

### Fonctionnels/E2E

- token valide → inscription autorisée ;
- token absent/invalide/expiré/réutilisé → refus sans écriture ;
- hostname/action incorrects → refus ;
- double clic/même `request_id` → une inscription et même résultat ;
- même clé avec payload différent → conflit ;
- Turnstile indisponible → comportement dégradé attendu ;
- `429` avec réponse UX exploitable ;
- aucun secret/token complet dans les logs.

Utiliser les clés officielles de test Turnstile, jamais les secrets prod. Référence :
[Cloudflare — testing](https://developers.cloudflare.com/turnstile/troubleshooting/testing/).

### Charge

- test L9.b technique avec bypass/clé de test contrôlé ;
- test final BIL avec Turnstile simulé de manière déterministe ;
- vérifier que le proxy PHP ne regroupe pas les 3 000 clients sous le même quota ;
- vérifier que Siteverify n'est pas le nouveau goulot ;
- NAT simulé : beaucoup de clients partageant une IP sans blocage abusif ;
- attaque simulée séparément, sans mélanger la preuve de capacité métier et la preuve anti-abus.

## 9. Découpage et estimation

| Lot | Livrable | Estimation |
| --- | --- | ---: |
| N-ANTI-0 | décision proxy/direct, menace, seuils initiaux et clés/env | 0,5 j |
| N-ANTI-1 | widget BIL + validation serveur Turnstile | 0,5–1 j |
| N-ANTI-2 | rate limits multi-signaux et correction identité IP/proxy | 0,5–1 j |
| N-ANTI-3 | idempotence `request_id` et statut de retry | 0,5–1 j |
| N-ANTI-4 | flags, métriques, alertes et runbook | 0,5 j |
| N-ANTI-5 | E2E, NAT partagé, mode dégradé et intégration k6 | 0,5–1 j |
| **Total** | MVP anti-bot/anti-abus LFD | **2,5–4,5 j-dev** |

## 10. Critères de sortie

- le CDC « CAPTCHA ou équivalent » est démontrable en recette ;
- aucun contournement par appel direct à l'endpoint public ;
- aucun blocage global causé par l'IP unique du proxy PHP ;
- NAT ministériel simulé sans faux positifs massifs ;
- double clic/retry sans double inscription ;
- modes `normal`, `event`, `attack` et dégradé testés ;
- secrets absents du front, des logs et du dépôt ;
- coût de latence mesuré et inclus dans les p95/p99 finaux ;
- runbook jour J utilisable par l'astreinte.

## 11. Hors scope LFD

- fingerprint matériel intrusif ;
- scoring antifraude/IA ;
- WAF sur mesure ;
- nouveau broker ;
- dashboard antifraude complet ;
- création dynamique de règles par le client ;
- garantie absolue « un appareil physique = une identité » sans authentification forte.
