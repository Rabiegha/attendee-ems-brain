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
- ✅ **Statelessness** = post-event, non-bloquant.
- ⏭️ **Prochaine action concrète :** coder L9 + test, puis **load test combiné** → relire ce doc au gate.
