# Points a considerer — 16 juillet 2026 matin

> Note de reprise avant de passer a la suite.  
> Objectif : ne pas perdre les dettes et decisions ouvertes vues pendant le rush.

---

## 1. Re-scan QR : l'action n'est pas encore loggee

### Constat

Le re-scan au meme point de controle n'est **pas logge** aujourd'hui.

Quand un QR deja scanne est presente de nouveau au meme point de controle, le back leve une
exception `ALREADY_CHECKED_IN`, mais rien n'est persiste pour garder la trace de l'action.

`AuditLog` existe dans Prisma, mais n'est pas appele dans ce chemin.

### Pourquoi c'est important

La regle metier voulue est : **garder l'action**, meme quand elle ne revalide pas l'entree.

Sans log, on perd :

- qui a rescane;
- quand le re-scan a eu lieu;
- ou / sur quel point de controle;
- la capacite a comprendre un incident terrain apres coup.

### Clarification metier

Le cycle de vie normal d'un QR autorise le **multi-scan legitime** entre plusieurs points de controle :

- entree event;
- session A;
- session B;
- checkout;
- re-checkin.

Donc "deja scanne" ne veut pas dire "QR invalide".

La vraie regle :

- multi-scan sur des points de controle differents = legitime;
- re-scan au **meme** point de controle = deja scanne, ne revalide pas, mais doit etre trace.

### Lien avec HMAC / chantier D

Le HMAC protege contre la falsification du QR :

```text
v1.<id>.<sig>
```

Il garantit que l'identifiant n'a pas ete modifie, mais il ne remplace pas un log metier.

HMAC = anti-falsification.  
Audit / log = trace de l'action terrain.

### Action a ajouter

- [ ] Ajouter le dev **H-D1** : persister le re-scan.
- [ ] Brancher `AuditLog` ou une table dediee `checkin_logs`.
- [ ] Couvrir le cas event **et** le cas session.
- [ ] Ajouter/adapter le test **H-T2** pour verifier que le re-scan est reellement persiste.

---

## 2. Dette infra : route `/jeu/` ajoutee manuellement dans nginx

### Constat

Le `location /jeu/` a ete ajoute **manuellement** au vhost nginx sur le VPS pour rendre le jeu
de warm-up accessible rapidement.

Ca marche a court terme, mais ce n'est pas encore une configuration perenne.

### Risque

La configuration nginx est versionnee dans le repo back, puis le VPS fait des checkouts lors des
deploys.

Donc si la route `/jeu/` reste uniquement ajoutee a la main sur le serveur, elle peut etre
**ecrasee au prochain CD prod**.

Effet possible :

- le jeu fonctionne maintenant;
- un deploy prod repasse la conf nginx a l'etat versionne;
- `/jeu/` disparait;
- le lien envoye dans les emails de warm-up peut tomber en 404.

### Decision a prendre

Si le jeu doit survivre au-dela de la campagne immediate, il faut le rendre versionne.

Sinon, accepter explicitement que c'est un ajout temporaire de campagne, avec le risque connu.

### Action si on veut le perenniser

- [ ] Ajouter la conf nginx dans le repo back, probablement dans `nginx/conf.d/`.
- [ ] Ajouter le service jeu dans `docker-compose.staging.yml`.
- [ ] Versionner le montage/persist de donnees du classement.
- [ ] Ajouter une verification simple apres deploy : `https://staging.attendee.fr/jeu`.
- [ ] Eviter les ajouts manuels non traces sur le VPS pour toute conf qui doit survivre a un checkout.

---

## 3. Comprehension de l'incident du 16/07 matin

### Lecture actuelle

L'incident du matin s'inscrit dans un pattern deja vu : une modification infra utile est faite
directement sur le VPS pour aller vite, mais la source de verite reste le repo.

Tant que le serveur tourne sans re-checkout/redeploy, la modif manuelle existe.

Mais des qu'un deploy repasse par le code versionne, toute conf non committee peut disparaitre.

### Cause racine probable

Cause racine process :

- changement fait a chaud sur le VPS;
- pas encore committe dans le repo;
- deploy/CD capable de re-checkout la conf versionnee;
- donc risque de divergence entre "ce qui tourne" et "ce qui est decrit dans Git".

Ce n'est pas un probleme de nginx en soi.  
C'est un probleme de **source de verite infra**.

### Impact a surveiller

- Route `/jeu/` et lien de warm-up.
- Vhosts nginx modifies manuellement.
- Services ajoutes hors `docker-compose*.yml`.
- Toute correction faite sur le VPS mais pas repercutee dans Git.

### Regle a retenir

Toute conf dont le serveur depend doit etre :

- soit temporaire et assumee comme jetable;
- soit versionnee dans les vraies branches qui alimentent le deploy.

Le raccourci acceptable en rush :

1. patch manuel pour debloquer;
2. test immediat;
3. note de dette;
4. commit de perennisation si la feature doit durer.

---

## Priorite conseillee

1. Ne pas bloquer la suite GCP/Gotenberg avec ces points.
2. Garder **H-D1 re-scan logge** comme dette fonctionnelle importante du chantier H.
3. Decider vite si `/jeu/` est jetable ou doit survivre.
4. Si le lien `/jeu/` a deja ete envoye a des utilisateurs, perenniser la route avant le prochain CD prod.
