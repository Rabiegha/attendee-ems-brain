# Stack staging — concepts infra (pgBouncer, pool, directUrl, vhost, env)

> **Date :** 2026-07-07 · **Contexte :** chantier A (refonte propre LFD 2026), rejeu du levier
> **L1/L2/L10** (stack staging isolée). Notions comprises en montant la stack — à garder au chaud.
> **Réfs :** [00-plan-action.md](../workstreams/en-cours/lfd2026/00-plan-action.md) ·
> [staging-setup.md](../infra/staging/staging-setup.md).

---

## 1. Ce que fait un vhost (virtual host)

Capacité d'**un seul serveur web** (nginx) à héberger **plusieurs domaines** sur la **même
machine / même IP**. À chaque requête, le navigateur envoie le domaine visé dans l'en-tête HTTP
`Host` ; nginx lit ce `Host` et choisit **quel bloc `server { }` (= quel vhost)** répond.

```
                                    ┌─ Host: attendee.fr          → vhost 1 → app prod
Requête → nginx (1 IP, port 443) ──┼─ Host: staging.attendee.fr  → vhost 2 → ems-staging-api
                                    └─ Host: logs.attendee.fr     → vhost 3 → app logs
```

Exemple de vhost staging (levier L15) :
```nginx
server {
    server_name staging.attendee.fr;                 # ce vhost gère CE domaine
    listen 443 ssl;
    ssl_certificate .../staging.attendee.fr/fullchain.pem;
    location /api { proxy_pass http://ems-staging-api:3000; }
}
```

**Analogie :** un immeuble (le serveur) = 1 adresse (l'IP). Le concierge (nginx) lit le nom sur
l'enveloppe (`Host`) et distribue au bon appartement (le bon vhost).

---

## 2. Pourquoi PAS de service nginx dans `docker-compose.staging.yml`

1. **Les tests de charge k6 tapent l'API en direct** (`127.0.0.1:3001` → API). On veut mesurer
   l'API, pas nginx. Une couche nginx fausserait la mesure.
2. **Le nginx du staging existe déjà ailleurs** : le nginx **de la prod** (`ems-nginx`) a reçu un
   **vhost `staging.attendee.fr`** (levier L15) qui proxy vers `ems-staging-api`. Donc pas besoin
   d'un 2ᵉ nginx dans la stack staging.

→ La stack staging est volontairement **minimale** (postgres + pgbouncer + redis + mailpit + api).

---

## 3. pgBouncer (L2) vs pool / `connection_limit` (L10)

Deux couches distinctes de gestion des connexions DB (pooler externe vs pool applicatif).
Détail complet → [2026-07-07-pgbouncer-et-pool-db.md](./2026-07-07-pgbouncer-et-pool-db.md).

---

## 4. `directUrl` Prisma (L3) — pourquoi deux URLs

pgBouncer en `transaction` casse les migrations → Prisma a une 2ᵉ URL directe.
Détail complet → [2026-07-07-directurl-prisma.md](./2026-07-07-directurl-prisma.md).

---

## 5. Variables d'env : template committé, secrets générés sur le serveur

**Deux fichiers, ne jamais confondre :**

| Fichier | Dans git ? | Contenu |
|---|---|---|
| `.env.staging.example` | ✅ committé | **placeholders** (`STAGING_PWD`, `CHANGE_ME_staging`…) |
| `.env.staging` | ❌ jamais (`.gitignore` `.env*`) | **vrais secrets**, uniquement sur le serveur |

Le vrai `.env.staging` est créé **sur le serveur** par `scripts/staging-make-env.sh` :
- génère des mots de passe aléatoires (`openssl rand`), remplace les placeholders, `chmod 600` ;
- **idempotent** : si `.env.staging` existe déjà, il n'y touche pas.

→ On committe le **template**, jamais les secrets.

### ⚠️ Piège : mot de passe Postgres figé dans le volume

Postgres **fige le mot de passe au tout premier démarrage**, à l'intérieur de son **volume de
données** (`ems_staging_postgres_data`). Il ne le relit **pas** au redémarrage.

Conséquence : si on **régénère** `.env.staging` (nouveau password aléatoire) tout en **gardant
l'ancien volume**, le nouveau password **ne matchera plus** celui gravé dans le volume → **l'API
n'arrive plus à se connecter à la base**.

**Règles :**
- régénérer l'env = soit **remettre le même password**, soit **repartir d'un volume vierge**
  (`docker compose -f docker-compose.staging.yml down -v`, qui supprime les volumes).
- c'est exactement pour ça que `staging-make-env.sh` est **idempotent** : il ne réécrit pas un
  `.env.staging` existant → il **protège** de ce désalignement password ↔ volume.

---

## 6. Rejouer L1/L2/L10 ne casse pas le staging déjà déployé

- Les fichiers rejoués sont **identiques** à ceux déployés (pris dans l'archive) → rien de
  structurel ne change.
- Le `.env.staging` du serveur **n'est pas dans git** → intouché.
- Redéploiement plus tard = `docker compose -f docker-compose.staging.yml up -d --build` :
  recrée seulement les conteneurs dont l'image/config a changé ; les **volumes nommés**
  (`ems_staging_postgres_data`…) sont **préservés** → aucune perte de données.
- **Pas besoin** de supprimer/reconstruire quoi que ce soit maintenant : ce rejeu = rangement
  d'historique git, pas un redéploiement.

---

## 7. Pourquoi ces leviers sont « simples à rejouer »

L1/L2/L3/L10 sont des leviers **d'infra/config** : fichiers **additifs et autonomes** (un YAML
compose, un template d'env, 5 lignes de schéma), **aucune logique métier touchée** → pas de
risque de régression. À l'inverse, **L9** (transaction d'inscription dans `public.service.ts`)
est de la **vraie chirurgie de code** → test de non-régression obligatoire.
