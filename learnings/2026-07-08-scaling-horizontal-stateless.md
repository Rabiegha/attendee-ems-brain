# Scaling horizontal : l'app doit être *stateless* d'abord

> **Date :** 2026-07-08 · **Contexte :** cadrage du chantier `api-scaling-clustering` (LFD 2026).
> **En une phrase :** ajouter un load balancer + des instances, c'est les 10 % faciles ; le vrai
> prérequis c'est que l'app ne garde **plus rien en mémoire vive** (état partagé en Redis).

---

## Scaling vertical vs horizontal

- **Vertical** = une **plus grosse** machine (plus de CPU/RAM). Simple, rapide, mais plafonné et
  point unique de panne.
- **Horizontal** = **plusieurs** machines derrière un load balancer. Élastique, résilient — mais
  ne marche **que si l'app est stateless**.

---

## Le vrai blocage : l'état en mémoire

Le load balancer + les instances multiples, c'est les **10 % faciles**. Le prérequis, c'est que
l'app soit **stateless** (aucun état en RAM d'un process). Et c'est là que ça coince ici :

l'API garde en **mémoire vive** : connexions socket.io, **présence des imprimantes**, état temps réel.

Si on lance 2 instances derrière un LB :

- une inscription arrive sur l'**instance A**,
- mais l'imprimante est « connue » de l'**instance B**,
- → **le badge ne s'imprime pas, en silence.**

> **C'est exactement pour ça que le cluster (L13) est interdit tant que ce n'est pas fait.**
> Ce n'est pas une limite de serveur, c'est une limite de **conception**. Rushé, ça casse
> l'impression temps réel **silencieusement** (pas d'erreur, juste des badges qui ne sortent pas).

```
❌ Aujourd'hui (état en mémoire)          ✅ Après externalisation
┌───────────────┐  ┌───────────────┐     ┌──────────┐   ┌──────────┐
│ Instance A    │  │ Instance B    │     │Instance A│   │Instance B│
│ imprimantes:  │  │ imprimantes:  │     └────┬─────┘   └────┬─────┘
│   1, 2        │  │   3, 4        │          │              │
└───────────────┘  └───────────────┘         └──────┬───────┘
        A ne voit pas B                              ▼
                                          ┌────────────────────────┐
                                          │ Redis (état partagé :   │
                                          │ présence, imprimantes)  │
                                          └────────────────────────┘
```

---

## Ce que « rendre stateless » veut dire concrètement

Dans l'ordre — **c'est du code applicatif, pas de l'infra** :

1. **socket.io redis-adapter** : les messages temps réel transitent par Redis (pas de process à process).
2. **Présence / imprimantes en Redis** : sorties de la RAM de chaque process.
3. **Sticky sessions côté nginx** : une session reste sur son instance le temps de sa vie.
4. **Tests cross-worker verts** : le **gate dur** — pas de cluster en prod tant qu'ils ne passent pas.

Une fois ça fait, ajouter des instances derrière un LB devient **trivial**.

---

## Le point clé : le blocage c'est le code, pas le cloud

Le scaling horizontal **ne dépend pas de GCP**. Une fois l'app stateless, on peut lancer 3 conteneurs
derrière **nginx sur OVH** aussi bien que sur GCP.

Ce que le cloud managé (GCP) apporte **en plus** : autoscaling (Cloud Run monte/descend selon la
charge), **Memorystore** (le Redis partagé managé), **LB managé**. C'est du **confort**, pas la
condition. **La condition, c'est le refacto d'externalisation de l'état — côté app.**

> Corollaire pour le call GCP : demander à l'équipe de **préparer l'infra** (Memorystore, LB,
> Cloud Run) est utile, mais **ça ne débloque pas le scaling** tant que l'état n'est pas sorti de
> la RAM. Ce refacto ne peut pas être délégué à l'infra.

---

## Pour l'event vs après

- **Event (4-5 sept)** : scaling **vertical** (plus gros VPS) + **offload Gotenberg** (sort le CPU
  PDF) + leviers de code. Sûr et rapide.
- **Après** : scaling **horizontal** = chantier `api-scaling-clustering`, fait au calme, avec le
  gate « tests cross-worker verts » avant toute mise en prod.
