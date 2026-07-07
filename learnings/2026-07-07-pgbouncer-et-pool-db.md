# pgBouncer (L2) vs pool / `connection_limit` (L10)

> **Date :** 2026-07-07 · **Contexte :** stack staging LFD 2026 (leviers L2 + L10).
> **En une phrase :** deux couches distinctes de gestion des connexions à PostgreSQL, dans
> **deux fichiers différents**.

---

## Le problème de fond

PostgreSQL gère **chaque connexion comme un processus** (avec sa RAM). Il ne peut pas en tenir
des milliers. Sous un pic (inscriptions, plusieurs workers), si l'appli ouvre trop de connexions
→ **Postgres sature** (le vrai goulot du scaling écriture).

---

## C'est quoi un « client » ici ?

Un **client** = **tout ce qui ouvre une connexion vers la base**. Ce n'est **pas** le navigateur
(il parle à l'API en HTTP, jamais à la DB). Le client de pgBouncer, c'est **l'API NestJS via
Prisma**.

```
Navigateur ──HTTP──▶ API NestJS (Prisma) ──connexions DB──▶ pgBouncer ──▶ PostgreSQL
   (pas un                  ▲                                  ▲
    client DB)         "les clients"                    peu de vraies
                       de pgBouncer                     connexions
```

Prisma maintient un **pool de connexions** ; **chaque connexion = un client** pour pgBouncer.
Quand l'appli scale (plusieurs process/workers), chacun a son propre pool → plus de clients →
c'est là que pgBouncer devient indispensable.

---

## Les deux couches (à ne jamais confondre)

| | **pgBouncer (L2)** | **pool / `connection_limit` (L10)** |
|---|---|---|
| C'est quoi | pooler **externe** devant Postgres | pool **interne de l'appli** (Prisma) |
| Rôle | garde peu de vraies connexions Postgres et les **recycle** entre tous les clients ; mode `transaction` = une connexion n'est prêtée que le temps d'**une transaction**, puis rendue | **plafonne** combien de connexions l'appli s'autorise à ouvrir |
| Réglé dans | **`docker-compose.staging.yml`** (service `pgbouncer`) | **`.env.staging`** (paramètre de `DATABASE_URL`) |
| Réglages | `POOL_MODE: transaction`, `DEFAULT_POOL_SIZE`, `MIN_POOL_SIZE`, `RESERVE_POOL_SIZE` | `?connection_limit=N` dans l'URL |
| Analogie | la réception qui fait passer 500 visiteurs par 3 portes | la règle « on n'envoie pas plus de N visiteurs à la fois » |

---

## Où se règle chaque limite (concret)

**Pool applicatif (L10)** — dans `.env.staging`, en paramètre de `DATABASE_URL` :
```dotenv
# défaut Prisma si absent = (nb_cpus × 2 + 1)
DATABASE_URL=postgresql://user:pwd@pgbouncer:6432/db?pgbouncer=true&connection_limit=20
#                                                                    ^^^^^^^^^^^^^^^^^^^ L10
```
(le `&` car c'est un 2ᵉ paramètre après `?pgbouncer=true`).

**Pool pgBouncer (L2)** — dans `docker-compose.staging.yml` :
```yaml
pgbouncer:
  environment:
    POOL_MODE: transaction
    DEFAULT_POOL_SIZE: "100"   # vraies connexions vers Postgres
    MIN_POOL_SIZE: "10"
    RESERVE_POOL_SIZE: "20"
```

---

## La règle à retenir

- « Combien de connexions **l'appli** ouvre » → `.env` (`connection_limit`) = **L10**.
- « Combien de vraies connexions **pgBouncer** garde vers Postgres » → compose (`DEFAULT_POOL_SIZE`) = **L2**.
- Les deux doivent rester **cohérents** : l'appli ne doit pas demander plus que ce que
  pgBouncer/Postgres peut tenir. Ensemble → on encaisse beaucoup de trafic sans écrouler Postgres.

---

## L'image mentale : un cabinet médical

- **Patients** = requêtes de l'appli (des milliers pendant un pic).
- **Médecins** = **vraies connexions Postgres** (chacune = un processus + RAM → chères, limitées).
- **Secrétaire qui régule la salle d'attente** = **pgBouncer**.

**Recycler** = garder peu de médecins (ex : 100) et les faire passer d'un patient au suivant :
dès qu'un patient a fini, le médecin est **libéré et redonné** au suivant. La même connexion
Postgres sert donc à plein de requêtes, l'une après l'autre.

**Mode `transaction`** = la finesse du recyclage : pgBouncer prête une vraie connexion Postgres
**uniquement le temps d'une transaction** (un bloc SQL indivisible, ex « insère l'inscription +
décrémente le quota »). Dès le `COMMIT`, il reprend la connexion et la donne au client suivant.
Le patient reste dans son fauteuil (connexion **vers pgBouncer**), mais le médecin (connexion
**vers Postgres**) ne lui est réservé que pendant l'acte. → **500 clients applicatifs** partagent
**100 vraies connexions**, car ils ne sont jamais tous en transaction au même instant.

---

## Oui, il y a bien DEUX pools (deux étapes du tuyau)

```
Appli NestJS/Prisma ──▶ [POOL A] ──▶ pgBouncer ──▶ [POOL B] ──▶ PostgreSQL
    (connection_limit)     L10        (DEFAULT_POOL_SIZE)  L2
```

Ce n'est pas « un pg / un db » : **les deux** concernent la base. La différence = **où** sur le
tuyau. Pool A = ce que l'appli ouvre vers pgBouncer (L10, `.env`). Pool B = ce que pgBouncer
ouvre vers Postgres (L2, compose).

---

## Pourquoi pool size (100) > limite (20) ?

Parce que `connection_limit=20` est **par process**, et on fait tourner **plusieurs process**
(scaling L8/L13). pgBouncer doit encaisser la **somme** de tous les pools A :

$$\text{DEFAULT\_POOL\_SIZE} \;\geq\; \text{connection\_limit} \times \text{nb process}$$

Ex : `100 ≥ 20 × (4-5 process)`. Si `DEFAULT_POOL_SIZE=20`, le **premier** process consomme tout
→ les autres attendent. Le 100 laisse de la marge + une réserve (`RESERVE_POOL_SIZE`) pour les pics.

---

## Cas surbooking : Σ pools A > pool B (ex : 4×100 = 400 vs 300)

- Tant que **< 300** transactions **simultanées** → aucun souci (99 % du temps, grâce au recyclage).
- La **301ᵉ** transaction alors que les 300 sont occupées → pgBouncer la **met en file d'attente**
  (il ne rejette pas tout de suite) :
  1. une connexion se libère à temps (`COMMIT`) → servie, attente de quelques ms, **invisible** ;
  2. rien ne se libère → au bout de `query_wait_timeout` (~120 s) → **erreur** (« pool exhausted »).
- Risque réel = sous **pic soutenu avec transactions longues**, la file grossit → **latence** puis
  **timeouts**. Le surbooking est un **pari** (« les 4 process ne seront pas tous à fond en même temps »).
- Léger surbooking = souvent OK et volontaire (comme une compagnie aérienne). Gros surbooking =
  dangereux. **Zéro attente garantie** ⟺ `DEFAULT_POOL_SIZE ≥ connection_limit × nb process`.
  À **valider par les tests k6** sur la stack staging.

---

## Coût d'un pool pgBouncer trop gros

Un gros `DEFAULT_POOL_SIZE` **ne se paie pas dans pgBouncer** (léger) mais **dans Postgres derrière** :

1. **RAM** : chaque vraie connexion = un processus (~5–10 Mo). 500 connexions = 2,5–5 Go **avant**
   toute requête → RAM prise au cache disque (`shared_buffers`) → Postgres **ralentit**.
2. **Contention CPU** : le serveur a N cœurs (~4). Trop de connexions **actives** se battent pour
   les mêmes cœurs → context switching → **débit qui baisse**. Courbe en **cloche** : le débit
   monte jusqu'à ~`nb_cœurs × 2-4` puis **redescend**.
3. **Faux sentiment de sécurité** : un gros pool masque un vrai problème (transaction longue,
   requête non optimisée), puis casse d'un coup, plus violemment.
4. **Borne dure** : `DEFAULT_POOL_SIZE` ≤ `max_connections` de Postgres (sinon refus de connexion).

```
Trop PETIT → file d'attente, latence, timeouts.
Trop GROS  → RAM gaspillée + CPU en surcharge → Postgres ralentit puis casse.
Bon chiffre ≈ nb_cœurs_Postgres × (2 à 4), à caler par tests k6.
```

Mettre un pool énorme **annule le bénéfice de pgBouncer** (on revient au « trop de connexions »).

---

## Le pool est ÉLASTIQUE (pas 100 connexions en permanence)

`DEFAULT_POOL_SIZE=100` est un **plafond**, pas un socle. Les 3 chiffres :

| Réglage | Rôle |
|---|---|
| `DEFAULT_POOL_SIZE` (100) | **plafond** : jamais plus de 100 vraies connexions |
| `MIN_POOL_SIZE` (10) | **socle** : ~10 gardées « au chaud » même à vide |
| `RESERVE_POOL_SIZE` (20) | **secours** : +20 débloquées si tout est occupé |

```
Aucune requête (nuit) ... ~10 ouvertes        (= MIN_POOL_SIZE)
Trafic normal ........... ~20-40 (à la demande)
Gros pic ................ jusqu'à 100 (+20 réserve)
Le pic retombe .......... pgBouncer referme le surplus → revient vers 10
```

pgBouncer ouvre les connexions **à la demande** et referme les inutilisées après un délai
d'inactivité. Garder un socle (10) évite de payer la latence d'ouverture (handshake TCP + auth +
init process) sur les **premières** requêtes d'un pic. → à vide, tu paies **~10**, pas 100.

---

## Où se paie la RAM ? Et en cloud managé ?

La RAM des connexions se consomme **sur la machine qui fait tourner PostgreSQL** (pgBouncer, lui,
est léger — il ne fait que router).

```
API ──▶ pgBouncer ──▶ PostgreSQL
        (léger)        ◀── la RAM des connexions se paie ICI
```

- **Stack staging actuelle** : Postgres = conteneur `ems-staging-postgres` sur **le même VPS** →
  RAM prise sur le VPS (d'où les caps `cpus`/`mem`).
- **Postgres cloud managé** (RDS, Supabase, Neon, DO…) : la RAM est chez le fournisseur, mais :
  - `max_connections` est **fixé par le tier** payé (petit tier = ~25–100 connexions seulement) ;
  - monter le plafond = **payer un tier supérieur** (coût RAM → coût €/mois) ;
  - beaucoup fournissent **déjà un pooler** (Supabase Supavisor, AWS RDS Proxy, Neon `-pooler`) →
    on utilise le leur au lieu de notre pgBouncer, même concept.
- **Piège cloud fréquent** : attaquer Postgres **en direct** (sans passer par le port pooler) →
  on épuise le petit `max_connections` du tier → erreurs « too many connections » au 1ᵉʳ pic.
  C'est exactement ce que L2/pgBouncer évite.
