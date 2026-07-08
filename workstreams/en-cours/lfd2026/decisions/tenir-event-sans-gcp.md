# Décision — Peut-on tenir l'event SANS GCP ? (+ gate de décision post-L9)

> **Objet :** trancher, à froid, la question la plus structurante du workstream : **peut-on absorber
> le pic d'inscriptions de LFD 2026 (MEAE, ~12 400 inscriptions, ~3 000 concurrentes à l'ouverture,
> 4-5 sept 2026) sur l'infra actuelle (VPS OVH), SANS attendre la migration GCP ?** La réponse
> conditionne : (1) le **focus du mois**, (2) le **discours à l'équipe**, (3) **clustering maintenant
> ou plus tard**, (4) le **sort du plan de continuité**.
> **Date :** 8 juillet 2026 · **Rattachement :** workstream `lfd2026`
> **Réfs :** [00-plan-action.md](../00-plan-action.md) (chantiers I/J/F) ·
> [01-suivi-leviers.md](../01-suivi-leviers.md) ·
> [carte stratégique 2 chantiers](../../../infra/lfd-2026-strategie-2-chantiers.md) ·
> [learning — pic inscription temps réel](../../../learnings/2026-07-08-pic-inscription-temps-reel.md) ·
> [ws 02 — capacité live forte charge](../../a-faire/sessions-inscriptions-lfd2026/02-capacite-live-forte-charge.md) ·
> `attendee-ems-back/docs/infra/lfd-2026-load-test-results.md` (mesures 25/06)

---

## 0. Réponse en une phrase

**Oui, on peut le faire sans GCP** — parce que le plafond ~33/s est **algorithmique (dans le code),
pas matériel (dans le serveur)**. GCP ne casse pas ce plafond ; **L9 le casse**. Migrer sur Cloud SQL
sans toucher à la transaction d'inscription = **rester à 33/s**. Donc : **L9 d'abord, mesure, et cette
mesure décide de tout le reste.**

---

## 1. Pourquoi ça stagne à ~33/s malgré « tous les leviers appliqués »

Les leviers déjà posés (**L1/L2/L10 — pgBouncer, pool, worker séparé**) ont réglé **l'isolation et
d'autres goulots potentiels**, mais **aucun n'a touché le vrai goulot**. Prouvé par nos mesures du 25/06 :

| Hypothèse testée | Mesure | Verdict |
|---|---|---|
| CPU | 2→4 cœurs = **32,8 → 32,65/s** (idem) | ❌ ce n'est pas le CPU |
| Pool DB | **1–4 connexions actives** sur 100 | ❌ ce n'est pas le pool |
| Email | worker séparé à **~8 % CPU** sous charge | ❌ ce n'est pas l'email |

**Le vrai goulot** = la **transaction Prisma d'inscription** (`registerToEvent`, `public.service.ts`)
qui **tient une connexion pendant un `COUNT` de capacité + upsert + create** → les inscriptions
concurrentes **se sérialisent**. On a blindé toute la maison **sauf la porte par laquelle tout le
monde passe**. C'est pour ça que le chiffre n'a pas bougé : **les leviers visaient à côté de la cible.**

---

## 2. Le seul levier qui casse les 33/s : **L9 (chantier I)**

**Transaction courte + compteur de capacité dénormalisé** :
`UPDATE … SET pris = pris + 1 WHERE pris < capacite` (atomique, ou Redis `DECR`) + transaction réduite
au **strict `create`** + garde d'unicité (contrainte, `P2002` → 409). Supprime la **sérialisation** →
le débit suit alors le **CPU** au lieu du verrou DB. **Gain plausible ×3 – ×10.**

> C'est **LE** levier. Tout le reste est secondaire. Une seule chose à faire ce mois pour le débit = **L9**.

---

## 3. Ce qui rend l'event tenable SANS GCP : le trio (tout sur le VPS actuel)

| Brique | Rôle | Besoin GCP ? |
|---|---|---|
| **L9** (tx courte + compteur) | casse le plafond 33/s → **100–300/s plausible** | ❌ non |
| **File BullMQ** | absorbe les **3 000 simultanés** → personne rejeté, la DB draine à son rythme | ❌ non (Redis déjà là) |
| **Portier Redis atomique** | anti-survente sur sessions à capacité serrée | ❌ non |

Avec ça, 3 000 inscriptions à l'ouverture drainent en **~10–30 s** derrière la file, sans saturer
Postgres. C'est le chantier **J** + le levier **L9**. **Aucune brique ne nécessite GCP.**

---

## 4. Les 4 décisions qui en découlent

### 4.1 GCP — découplé de l'event
GCP = **cible durable** (HA + PITR + autoscaling managés), **pas un prérequis** pour tenir septembre.
Le mur est dans la transaction, pas dans le serveur → on le règle sur l'infra actuelle.

### 4.2 Clustering — **plus tard, et sous condition mesurée**
Bloqué de toute façon par le **pas-encore-stateless**, ajoute de la complexité (état partagé
sockets+imprimantes), et **une instance unique post-L9 couvre très probablement l'event** (3 000
derrière une file à 100–300/s = quelques dizaines de secondes). → **Ne pas clusteriser maintenant.**
Le clustering est une réponse à un **manque mesuré**, pas à une intuition.

### 4.3 Continuité — ne « tombe pas à l'eau », elle se **coupe en deux**
- **HA (rester debout)** = réplication + bascule → **besoin d'un 2ᵉ nœud** → **reporté à GCP** (Cloud
  SQL natif). *(C'est bien ça que « il faut un autre serveur » signifiait.)*
- **Anti-perte / corruption** = **backup MVP** (dump quotidien + **offsite R2** + restore testé) +
  PITR léger sur le VPS → **part HORS du serveur** → **faisable sans GCP, maintenant** (chantier E).

Le vrai risque institutionnel MEAE = **perdre les données**, pas 5 min d'indispo. **Cette moitié-là est
couverte sur le VPS.** La contrainte « même serveur » bloque **l'HA**, pas le **backup**.

> #### 🔌 HA : comment marche VRAIMENT le failover (et pourquoi 1 VPS ne peut pas)
>
> **Le problème :** si Attendee/DNS pointe sur **l'IP fixe de la machine**, quand elle meurt l'IP meurt
> avec elle → **failover impossible**. La HA consiste à **ne jamais pointer sur l'IP de la machine**, mais
> sur **un point stable qu'on peut re-brancher** sur un autre nœud. 3 façons :
>
> 1. **IP flottante** (OVH « IP Failover ») : une IP publique **détachable/rattachable via API**. Les clients
>    pointent dessus ; quand le nœud A meurt, un orchestrateur la **rebranche sur B**. Même IP, autre machine.
> 2. **Load balancer devant** : le DNS pointe sur le **LB** (IP stable) ; il **health-check** A et B et ne
>    route qu'aux nœuds sains. A tombe → LB bascule sur B automatiquement.
> 3. **Bascule DNS** (le moins bon) : réécrire l'enregistrement DNS vers B — ⚠️ **cache TTL** → lent/peu fiable.
>
> **Côté DB :** l'app parle au **primaire courant** via un proxy/VIP piloté par un orchestrateur (**Patroni**…).
> En **managé (Cloud SQL)** : un **seul endpoint stable**, GCP le re-pointe tout seul → aucune IP à gérer.
>
> **Qui déclenche :** un health-check qui (1) détecte le primaire mort, (2) **promeut** le standby, (3) **déplace
> l'IP flottante / met à jour le LB**. Automatique (Patroni/keepalived/managé) ou manuel (script/bouton).
>
> **Le piège maison — split-brain :** si mal fait, **les 2 nœuds se croient primaires** → divergence. Il faut un
> **arbitre/quorum** (3ᵉ témoin). C'est **la partie difficile** → raison n°1 de préférer le managé GCP.
>
> **Notre situation :** 1 seul VPS, **pas d'IP flottante, pas de LB, pas de standby** → **aucun failover auto**.
> Si la machine meurt = **manuel** (monter un serveur + **restaurer le backup** + re-pointer l'IP) = **recovery**,
> pas HA. C'est **pour ça que l'HA attend GCP** (endpoint + LB + quorum managés) ; pour l'event, le filet =
> **backup offsite + restauration rapide** (RTO borné, suffisant face au vrai risque = perte de données).

> #### 🎯 Décision nette : PAS de HA pour l'event — risque assumé
> Si la machine meurt jour-J → **down + recovery manuel** depuis backup (RTO borné, minutes → 1-2 h).
> C'est un **choix**, pas un oubli : la HA couvre le risque **le moins probable** (panne matérielle sur
> 2 jours) et **ne couvre PAS** les vrais dangers (saturation, disque, perte de données).
> **⚠️ À vérifier avant de valider :** **pas de SLA d'uptime contractuel** imposé par le MEAE. Si SLA strict
> → accélérer GCP **ou** failover OVH temporaire pour les 2 jours (seul cas qui changerait la décision).

> #### 🗺️ Carte des vrais risques event (tous couverts SANS GCP)
>
> | Risque | Probabilité | La HA aide ? | Parade réelle (sans GCP) |
> |---|---|---|---|
> | **Saturation CPU/DB** (3000 concurrents) | **élevée** | ❌ | **L9 + file BullMQ** |
> | **Disque plein** | **moyenne-élevée** | ❌ (réplica se remplit pareil) | **0-MON alerte disque + capacité + rotation logs + offload R2** |
> | **Perte / corruption données** | moyenne | ❌ (réplique la bêtise) | **backup offsite + restore testé** |
> | **Machine meurt** (hardware) | **faible** | ✅ | *(HA → reportée GCP, risque assumé)* |
>
> **La HA ne couvre que la dernière ligne — la moins probable. Les 3 premières = les vrais dangers, réglés sur le VPS.**

> #### 💽 Zoom « disque plein » (mode de panne sous-estimé)
> **Pourquoi ça met l'API à terre :** disque plein → **Postgres ne peut plus écrire son WAL** → refuse
> les écritures / crash → **API down**. Coupables typiques : logs non tournés (le `console.log` par
> inscription !), **WAL accumulé** (slot de réplication bloqué = classique), bloat MVCC sous pic
> d'`UPDATE`, PDF/backups stockés en local, Redis persistant + gros backlog.
> **La HA n'aide pas** (voire empire : un runaway remplit primaire **et** réplica). **Parades :**
> (1) **alerte disque 70/80/90 %** *(la n°1)*, (2) **marge de capacité** dimensionnée pour l'event,
> (3) **rotation des logs**, (4) **gestion WAL / slots**, (5) **offload PDF+backups → R2**,
> (6) **autovacuum réglé**, (7) **runbook « libérer / redimensionner le volume à chaud »** (OVH).
> → C'est du ressort de **0-MON + hygiène**, pas de la HA. Consolidé dans le workstream **résilience event**.

### 4.4 Statelessness — **non-bloquant pour l'event**
Prérequis du **clustering / GCP**, pas du plan « 1 VPS + L9 + file ». → travail **post-event / migration**,
ne pas le laisser bloquer le plan event.

---

## 5. Gate de décision post-L9 (le point qui tranche tout)

```
1. Coder L9  (feat/register-tx-slim)  + test de non-régression (anti-survente préservé)
2. Load test COMBINÉ sur staging + k6 : inscriptions + check-ins simultanés
3. Lire le débit soutenu obtenu :
   ├─ ≥ cible event  → instance unique suffit
   │                   → clustering REPORTÉ post-event / GCP
   │                   → GCP = cible durable, pas prérequis
   └─ < cible event  → alors seulement : envisager clustering (après statelessness)
```

> ⚠️ **Statut honnête :** ceci est une **hypothèse très solide**, pas une **preuve**. La preuve = le
> **load test combiné APRÈS L9**. Tant que L9 n'est pas mesuré, on dit « très probablement oui »,
> pas « oui certifié ». Le test est **rapide** et tranche **les 3 questions d'un coup** (débit,
> clustering, GCP).

**À définir avant le test :** la **cible event chiffrée** (ex. « drainer 3 000 inscriptions en < X min
est acceptable »), sinon le gate n'a pas de seuil.

---

## 6. Discours à l'équipe (résumé froid pour le call)

1. **Le plafond 33/s est un bug de code, pas de matériel** → réglé sur l'infra actuelle par **L9**.
2. **Plan event = L9 + file BullMQ + portier Redis** → on tient l'event **sur le VPS**.
3. **GCP est découplé de l'event** : cible durable (HA/PITR/autoscaling), **pas un prérequis**.
4. **Clustering reporté**, décidé **après mesure** de L9.
5. **Continuité** : backup offsite **maintenant** (anti-perte) ; **HA avec GCP** plus tard.

---

## 7. Décision actée (2026-07-08)

- ✅ **On tient l'event sans GCP.** Focus du mois = **L9** (débit) puis **J** (file + portier Redis).
- ✅ **Clustering NON maintenant** → gate mesuré post-L9.
- ✅ **Continuité** = backup MVP now (VPS) ; HA reportée à GCP.
- ✅ **Pas de HA event = risque assumé** (à confirmer : pas de SLA MEAE strict).
- ✅ **Statelessness** = post-event, non-bloquant.
- ⏭️ **Prochaine action concrète :** coder L9 + test, puis **load test combiné** → relire ce doc au gate.
- 🧾 **Vérif finale avant J-7 :** [workstream résilience event](../../a-faire/resilience-event-lfd2026/README.md) — checklist que **tout ce qui protège** (saturation / disque / perte / recovery) est en place et **testé**.
