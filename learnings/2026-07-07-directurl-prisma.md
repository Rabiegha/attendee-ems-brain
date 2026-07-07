# `directUrl` Prisma (L3) — pourquoi deux URLs de base

> **Date :** 2026-07-07 · **Contexte :** stack staging LFD 2026 (levier L3).
> **En une phrase :** avec pgBouncer en mode `transaction`, Prisma a besoin d'une **2ᵉ URL** qui
> attaque Postgres en direct pour les **migrations**.

---

## Le problème

pgBouncer en mode `transaction` est parfait pour les requêtes normales… mais il **casse les
migrations Prisma**. Les migrations utilisent des fonctionnalités **niveau session** (verrous
d'avis / advisory locks, requêtes préparées, transactions longues) que le mode `transaction`
ne supporte pas (une connexion n'y est prêtée que le temps d'une transaction).

---

## La solution : deux URLs

| Variable Prisma | Env | Va vers | Sert à |
|---|---|---|---|
| `url` | `DATABASE_URL` | pgBouncer (port **6432**) | requêtes **normales** de l'appli (runtime) |
| `directUrl` | `DIRECT_URL` | Postgres **direct** (port **5432**) | **migrations** / introspection uniquement |

Dans `.env.staging.example` :
```dotenv
DATABASE_URL=postgresql://user:pwd@pgbouncer:6432/db?pgbouncer=true   # runtime → via pgBouncer
DIRECT_URL=postgresql://user:pwd@postgres:5432/db                     # migrations → direct
```

---

## Le levier L3 = ~5 lignes

Ajouter `directUrl` au bloc `datasource` de `prisma/schema.prisma` :
```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")   // ← ajouté par L3
}
```

À partir de là, Prisma route **tout seul** : les `prisma migrate` passent par `DIRECT_URL`, le
runtime par `DATABASE_URL`.

---

## Pourquoi c'est « simple à rejouer »

C'est un levier **d'infra/config** : une addition de **5 lignes** dans le schéma, **aucune
logique métier touchée** → zéro risque de régression. C'est l'inverse d'un levier comme **L9**
(chirurgie de code dans `public.service.ts`, test de non-régression obligatoire).
