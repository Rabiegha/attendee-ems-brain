# Leviers eventuels capacite — si L9a/L9b/L9.1 ne suffisent pas

> Note de cadrage ajoutee le 16/07/2026.
> Statut : **a explorer / a considerer**, pas un engagement de scope immediat.
> Objectif : garder sous la main les plans B si le plafond inscription reste trop bas apres L9b/L9a/L9.1.

## Probleme vise

Le plafond observe tourne autour de ~30 inscriptions/s selon les mesures historiques et le cadrage
LFD. L9b, L9a et L9.1 restent les premiers leviers a faire parce qu'ils reduisent le travail critique par
inscription/scan :

- **L9b** : transaction d'inscription session plus courte (`registerToSession`), prioritaire LFD.
- **L9a** : transaction d'inscription event plus courte (`registerToEvent`), a garder pour les formulaires event classiques.
- **L9.1** : compteur session/presence en O(1), pour eviter les `COUNT(*)` qui grossissent avec le volume.

Si ces deux leviers ne donnent pas assez de marge, ne pas improviser pendant le rush : choisir dans les
leviers ci-dessous, mesurer sur staging avec k6, puis seulement deployer.

## Leviers eventuels a explorer

### 1. Portier Redis inscription/capacite

But : reserver la capacite dans Redis avant l'ecriture Postgres, avec reconciliation ensuite.

- Impact attendu : protege Postgres et evite la survente sous pic.
- Utilite : forte pour billetterie publique, sessions a jauge, ouverture massive.
- Risque : il faut une reconciliation propre Redis -> Postgres en cas d'erreur apres reservation.
- Lien : chantier J capacite live.

### 2. Cache lecture public

But : servir l'etat public `ouvert / presque complet / complet` depuis Redis ou cache court, pas depuis
Postgres a chaque affichage.

- Impact attendu : reduit les lectures parasites pendant que Postgres encaisse les ecritures.
- Utilite : tres forte si 1000+ visiteurs rafraichissent la page de billetterie.
- Risque : etat legerement stale acceptable, mais il faut que l'ecriture reste la source anti-survente.

### 3. Queue / differer les effets secondaires

But : garder l'inscription synchrone minimale et deferer ce qui n'est pas indispensable a la reponse HTTP.

Candidates :

- email deja largement couvert par C2 (`EMAIL_QUEUE_ENABLED=true`) ;
- webhooks externes ;
- scoring / affinites ;
- statistiques non critiques ;
- tracking Mailgun `opened` si bruit DB : webhook -> queue `email.events` -> batch/dedup/agregation.

Impact attendu : baisse de latence et de contention pendant le pic.

### 4. Mode degrade event

But : prevoir des interrupteurs simples, activables sans redeploy.

Exemples :

- ralentir ou pauser l'envoi email via throttle Redis ;
- desactiver temporairement les stats live non critiques ;
- reduire les recalculs/metriques post-inscription ;
- afficher une jauge publique approximative plutot que temps reel exact.

Impact attendu : permet de sauver le flux inscription si un service secondaire devient bruyant.

### 5. File d'attente / limitation cote public

But : lisser l'arrivee au lieu de laisser 3000 personnes taper le submit en meme temps.

Options :

- ouverture par lots ;
- cooldown front apres tentative ;
- rate limit par IP plus intelligent ;
- page d'attente simple si saturation.

Impact attendu : transforme un pic brutal en debit soutenable.

### 6. Infra seulement apres reduction du travail par inscription

But : augmenter CPU/RAM/Postgres/pool seulement apres L9b/L9a/L9.1 et mesure.

- Impact attendu : utile si le code est deja suffisamment maigre.
- Risque : si le goulot est une transaction longue ou un lock, ajouter de l'infra aide peu.
- Note : GCP/Cloud SQL reste une option post-event ou si la mesure prouve que le VPS est le plafond.

## Version livrable en 10 jours — paquet de securisation

Si on decide de faire une version capacite en 10 jours, ne pas viser tous les leviers. Viser un paquet
court, mesurable, deployable :

1. **J1-J2 — L9b + L9a**
   - Slim transaction inscription session puis event.
   - Tests non-regression : inscription identique, anti-doublon, capacites event/session respectees.
   - k6 avant/apres sur staging.

2. **J3-J4 — L9.1**
   - Compteur presence/session O(1).
   - Tests : IN/OUT, double-scan, annulation, occupation correcte.
   - k6 scan/session si possible.

3. **J5-J6 — Portier Redis minimal**
   - Reservation capacite atomique Redis pour les flux publics a jauge.
   - Reconciliation simple en cas d'echec DB.
   - Feature flag pour pouvoir desactiver.

4. **J7 — Cache lecture public**
   - Etat public cache-first : ouvert / presque complet / complet.
   - TTL court + invalidation sur ecriture.

5. **J8 — Mode degrade**
   - Flags operationnels : pause email, baisse throttle, desactivation stats non critiques.
   - Runbook court : quoi couper dans quel ordre.

6. **J9 — Load test combine**
   - k6 inscription + scan/session + webhooks/email en bruit de fond.
   - Mesurer p95, taux erreur, CPU, Postgres, Redis, queue email.

7. **J10 — Stabilisation + decision**
   - Fix des regressions.
   - Decision go/no-go prod.
   - Documenter le seuil soutenable et les boutons d'urgence.

## Regle de decision

- Si L9b/L9a/L9.1 donnent une marge confortable au test k6 final : ne pas ajouter de complexite.
- Si le plafond reste trop proche du besoin event : priorite au **portier Redis + cache public**.
- Si la DB tient mais les effets secondaires bruitent : priorite aux **queues / mode degrade**.
- Si CPU/infra devient clairement le plafond apres code maigri : envisager scale infra/GCP.
