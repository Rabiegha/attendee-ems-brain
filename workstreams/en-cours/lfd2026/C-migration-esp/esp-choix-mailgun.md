# Choix ESP — Décision : **Mailgun** (vs Brevo / Scaleway / Postmark) — LFD 2026

> **Objet :** trancher le fournisseur d'envoi d'emails (ESP) pour l'événement La Fabrique de la
> Diplomatie 2026 (MEAE, ~12 400 inscriptions, 4-5 sept). Aujourd'hui Attendee envoie en
> **nodemailer SMTP direct** (pas d'ESP) → risque de délivrabilité majeur sur un envoi de masse.
> **Comparatif :** 1er juillet 2026 · **✅ Décision : 11 juillet 2026 → Mailgun** · workstream `lfd2026`
> **Réfs :** [diagnostic email/billet/wallet](../B-email-billet-pdf/email-billet-wallet.md) (§1 stack email, §6 Q2) ·
> [00-plan-action.md](../00-plan-action.md) (chantier C1/C2) ·
> [rapport source (PDF)](./ChoixESP-transactionnel-pour-Attendee-LFD.pdf) ·
> [setup/warm-up (C1)](./MAILGUN_WARMUP_PLAN.md) · [migration technique (C2)](./mailgun-migration-technique.md) ·
> [warm-up opérationnel (C3)](./c3-warmup-delivrabilite.md)
>
> ⚠️ **Les tarifs ci-dessous sont des ordres de grandeur** — à revérifier sur les sites des
> fournisseurs au moment du choix (les grilles changent souvent).

---

## 0. ✅ Décision actée (11/07/2026) — **Mailgun EU, IP partagée**

**Retenu :** **Mailgun**, région **EU** (`api.eu.mailgun.net`), **IP partagée**, **API HTTP** +
**queue BullMQ** + **webhooks**. Sous-domaine d'envoi : **`mail.attendee.fr`**.

### Pourquoi Mailgun (et pas Brevo/Scaleway/Postmark)

- 🟢 **Résidence UE explicite et documentée** : Mailgun permet de choisir la région à la création
  du domaine ; message data / logs / suppressions / stats / IP restent **« region-bound » EU**.
  C'est l'argument **souveraineté MEAE** le plus solide _publiquement_ (Brevo = éditeur FR mais
  résidence des message data moins nette ; Postmark = **US only**, disqualifié).
- 🟢 **IP partagée assumée** : Mailgun recommande officiellement le **shared** pour < 100k/mois ou
  volume volatil = exactement notre profil → **pas d'IP dédiée**.
- 🟢 **Burst ~3 000** géré par **queue applicative + API HTTP** (plus fiable que SMTP à l'échelle).
- 🟢 **Coût bas** (~$33 Basic / ~$35 Foundation) — non discriminant.
- 🟡 **Warm-up quand même** : même en IP partagée, on **chauffe le domaine** `mail.attendee.fr`
  (~2 sem min, 3-4 idéal) → voir [MAILGUN_WARMUP_PLAN.md](./MAILGUN_WARMUP_PLAN.md).
- ✅ **C1 terminé le 15/07** : accès OVH obtenu, plan acheté, DNS/vérification Mailgun OK.
  Le risque restant est le warm-up réel / délivrabilité → suivi dans C3.

### Comparatif compact (source : rapport PDF)

| Critère                     | **Mailgun** 🥇                                 | **Brevo** 🥈                                              | **Postmark** ❌                  |
| --------------------------- | ---------------------------------------------- | --------------------------------------------------------- | -------------------------------- |
| Résidence UE (message data) | 🟢 Région EU explicite, region-bound           | 🟠 Éditeur FR mais opérations UE+US, transferts possibles | 🔴 US only, **pas de projet UE** |
| Souveraineté MEAE           | 🟢 La plus défendable _publiquement_           | 🟢 Meilleure _narration française_                        | 🔴 Faible                        |
| Burst 3000 / débit          | 🟢 API HTTP + queue (Rapid Fire en enterprise) | 🟢 1 000 RPS documenté                                    | 🟢 batch 500/appel               |
| IP                          | 🟢 **Partagée** reco < 100k/mois               | 🟠 Dédiée = Enterprise only                               | 🟢 Partagée reco                 |
| Coût ~20k/2j                | 🟢 ~$33-35/mois                                | 🟠 « qq dizaines €/mois » à confirmer                     | 🟡 ~$30/mois                     |
| DX Node/NestJS              | 🟢 SDK `mailgun.js` (EU url)                   | 🟢 `@getbrevo/brevo`                                      | 🟢 la meilleure                  |
| Onboarding                  | 🟢 DNS + vérif domaine                         | 🟠 approbation d'envoi requise                            | 🟢 rapide                        |

> **Plan B = Brevo** si le critère dominant devient « éditeur **français** » (obtenir alors par écrit
> la localisation effective des message data + validation onboarding + support pendant le pic).
> **Postmark** = plan B technique seulement si la souveraineté est requalifiée en simple conformité RGPD.

> 📌 **Le comparatif détaillé ci-dessous (Brevo / Scaleway / SES / Mailjet / Postmark) est conservé
> pour historique et traçabilité de la décision.** La recommandation §5 est **superseded par §0**.

---

## 1. Le besoin réel

- **Type d'email :** **transactionnel** (billet + QR + PDF joint), 1 email par inscription,
  déclenché à la confirmation. Pas du marketing/newsletter.
- **Contrainte forte MEAE :** client **institutionnel** → **souveraineté / hébergement UE (France
  de préférence)**, conformité RGPD, réputation d'expéditeur maîtrisée.
- **Intégration actuelle :** `nodemailer` SMTP direct → migration la moins coûteuse = **relais SMTP**
  d'un ESP (changer host/port/credentials), l'API est optionnelle.
- **Pièce jointe :** le billet **PDF doit être joignable** (SMTP = OK partout ; via API = vérifier
  la limite de taille des attachments).

### 🎯 Profil de charge réel (le critère qui prime)

> Le vrai dimensionnant **n'est PAS le volume mensuel** (faible), mais le **débit en rafale** :
> un pic d'inscriptions génère un pic d'emails **quasi simultané**.

| Métrique                            | Valeur                                                     | Conséquence ESP                                            |
| ----------------------------------- | ---------------------------------------------------------- | ---------------------------------------------------------- |
| **Inscriptions simultanées au pic** | **~3 000 concurrentes**                                    | ~3 000 emails déversés en rafale dans la file              |
| **Volume total événement**          | **~20 000 sur 2 jours** (4-5 sept)                         | quota journalier à couvrir : ~10-15k/jour                  |
| **Débit d'envoi requis**            | drainer 3 000 mails avec **PDF joint** en quelques minutes | il faut **10-20 emails/s soutenus** pour vider en ~3-5 min |
| **Payload**                         | email **+ PDF billet** (attachment)                        | débit effectif plus bas que du texte seul                  |

**Ce que ça impose à l'ESP :**

1. **Débit / rate-limit** suffisant (emails/s et emails/h) — **pas** juste un quota mensuel.
2. **Quota journalier** ≥ ~15 000/jour **sans throttling** ni passage en revue manuel.
3. **Absorption des rafales** : un compte neuf / une IP dédiée froide **ne tient PAS 3 000 mails
   d'un coup** sans warm-up → soit **IP mutualisée** (pool déjà chaud), soit **warm-up planifié**.
4. **File tampon côté Attendee** (BullMQ déjà en place) → on **lisse** le pic si l'ESP plafonne :
   les mails partent au débit max de l'ESP, la file absorbe le reste (billet reste dispo via lien R2).

> ⚠️ **Risque n°1 = throttling/blocage sur rafale**, pas le coût. Un ESP avec un quota d'entrée
> bas ou une IP dédiée non chauffée **jettera ou différera** une partie des 3 000 emails du pic.

---

## 2. Critères de choix (pondérés pour MEAE)

| Critère                                                  | Poids       | Pourquoi                                                         |
| -------------------------------------------------------- | ----------- | ---------------------------------------------------------------- |
| **Débit / rate-limit en rafale (emails/s)**              | 🔴 **Fort** | **3 000 inscriptions simultanées** → pic d'emails à drainer vite |
| **Quota journalier sans throttling**                     | 🔴 **Fort** | **~20 000 sur 2 jours** (~10-15k/jour)                           |
| **Souveraineté / hébergement UE-FR**                     | 🔴 Fort     | Client institutionnel MEAE, données nominatives                  |
| **Délivrabilité transactionnelle**                       | 🔴 Fort     | Un billet non reçu = incident visible                            |
| **Gestion des rafales (IP mutualisée chaude / warm-up)** | 🔴 Fort     | Une IP dédiée froide ne tient pas 3 000 mails d'un coup          |
| **Facilité d'intégration (SMTP nodemailer)**             | 🟠 Moyen    | Migration mini-effort                                            |
| **Séparation transac / marketing**                       | 🟠 Moyen    | Ne pas polluer la réputation transac                             |
| **Tarif**                                                | 🟢 Faible   | Volume faible → non discriminant                                 |
| **Observabilité (logs, webhooks, bounces)**              | 🟠 Moyen    | Suivi des échecs d'envoi                                         |

---

## 3. Tableau comparatif

| Critère                                        | **Brevo** (ex-Sendinblue)                                                                                                                                                                                                                                                        | **Scaleway TEM**                                                                             | **Amazon SES**                                                                                                            | **Mailjet** (Sinch)                                                                                                               | **Postmark** (ActiveCampaign)                                      |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Origine / hébergement**                      | 🇫🇷 France, données UE                                                                                                                                                                                                                                                            | 🇫🇷 France (souverain, Paris)                                                                 | 🇺🇸 US (régions UE dispo : Paris/Francfort)                                                                                | 🇫🇷 origine FR (Sinch), UE dispo                                                                                                   | 🇺🇸 US (pas de région UE claire)                                    |
| **Souveraineté MEAE**                          | 🟢 Très bon                                                                                                                                                                                                                                                                      | 🟢 **Excellent** (cloud souverain FR)                                                        | 🟠 Correct si région UE + garanties                                                                                       | 🟢 Bon                                                                                                                            | 🔴 Faible                                                          |
| **SMTP relay (nodemailer)**                    | 🟢 Oui                                                                                                                                                                                                                                                                           | 🟢 Oui                                                                                       | 🟢 Oui                                                                                                                    | 🟢 Oui                                                                                                                            | 🟢 Oui                                                             |
| **API transactionnelle**                       | 🟢 Oui                                                                                                                                                                                                                                                                           | 🟢 Oui                                                                                       | 🟢 Oui                                                                                                                    | 🟢 Oui                                                                                                                            | 🟢 Oui (excellente)                                                |
| **Débit / rafale 3000**                        | 🟢 **Confirmé** : pic vidé en ~3 s (goulot = délivrabilité, pas l'API)                                                                                                                                                                                                           | 🟠 **À valider** (quota compte à relever)                                                    | 🟢 Élevé (hors sandbox)                                                                                                   | 🟠 Élevé mais non chiffré                                                                                                         | 🟢 Élevé, illimité/jour                                            |
| **Rate-limit (RPS) — source doc**              | 🟢 **1 000 RPS** (`POST /v3/smtp/email`, tous plans ; 2 000 Pro, 6 000 Enterprise) [doc](https://developers.brevo.com/docs/api-limits)                                                                                                                                           | 🟠 **Non publié** → quota compte à demander au support                                       | 🟠 **1 email/s en sandbox** ; ajustable (élevé) hors sandbox [doc](https://docs.aws.amazon.com/ses/latest/dg/quotas.html) | 🟠 « high rate limit » transac, **non chiffré publiquement** [doc](https://dev.mailjet.com/email/reference/overview/rate-limits/) | 🟢 Élevé ; **limite journalière illimitée** (non chiffré RPS)      |
| **Quota journalier (~15k/j)**                  | 🟢 OK selon plan                                                                                                                                                                                                                                                                 | 🟠 **À confirmer** vs pic                                                                    | 🟢 OK après sortie sandbox                                                                                                | 🟢 OK selon plan                                                                                                                  | 🟢 OK (illimité/jour)                                              |
| **Délivrabilité transac**                      | 🟢 Bonne (99 % annoncé), IP mutualisée chaude                                                                                                                                                                                                                                    | 🟢 Bonne ; **SLA 99,9 %** (plan Scale) + score réputation domaine                            | 🟢 Excellente si SPF/DKIM/DMARC + warm-up                                                                                 | 🟢 Bonne ; services délivrabilité en option                                                                                       | 🟢 **Référence du marché** (ex. +11 % open rate vs SES chez Asana) |
| **IP dédiée**                                  | 🟢 Option (plans sup.)                                                                                                                                                                                                                                                           | 🟠 À vérifier (surtout IP mutualisées)                                                       | 🟢 Option                                                                                                                 | 🟢 Option                                                                                                                         | 🟢 Option (gros volumes)                                           |
| **Séparation transac/marketing**               | 🟠 Plateforme mixte                                                                                                                                                                                                                                                              | 🟢 **100% transac**                                                                          | 🟢 100% transac                                                                                                           | 🟠 Mixte                                                                                                                          | 🟢 100% transac (streams séparés)                                  |
| **Webhooks bounces/opens**                     | 🟢 Oui                                                                                                                                                                                                                                                                           | 🟢 Oui                                                                                       | 🟢 Oui (SNS)                                                                                                              | 🟢 Oui                                                                                                                            | 🟢 Oui                                                             |
| **Tarif (ordre de grandeur)**                  | Freemium + paliers                                                                                                                                                                                                                                                               | 🟢 **Très bas** (~qq € pour ce volume)                                                       | 🟢 **Le moins cher** (~0,10 $/1000)                                                                                       | Freemium + paliers                                                                                                                | 💰 Plus cher (premium)                                             |
| **Tarif ~30k emails / 2 j (marge) + pic 3000** | **2 options** : (A) **Abonnement** Starter+ palier 30k ≈ ~€30/mois, **aucune limite/jour**, mais volume **expire chaque mois** ; (B) **Crédits prépayés (pay-as-you-go)** → **aucune limite/jour, sans expiration** (prix exact au curseur in-app) — RPS 1 000 dans les deux cas | 🟢 **~€7,5** (Essential PAYG : 300 gratuits + ~29,7k × €0,25/1000) ⚠️ quota compte à relever | 🟢 **~3 $** (0,10 $/1000 × 30k) ⚠️ + sortie de sandbox à anticiper                                                        | ~$25-35/mois (palier ~30k au curseur)                                                                                             | 💰 **~$42/mois** (Pro : 10k à $16,50 + 20k × $1,30/1000)           |
| **Complexité setup**                           | 🟢 Simple                                                                                                                                                                                                                                                                        | 🟢 Simple                                                                                    | 🔴 Plus technique (IAM, SNS, région)                                                                                      | 🟢 Simple                                                                                                                         | 🟢 Simple                                                          |

> Légende : 🟢 fort · 🟠 moyen/à vérifier · 🔴 faible · 💰 cher.
> **Cadrage tarifaire :** tous les prix « ~30k / 2 j » sont dimensionnés pour **~30 000 emails** (marge
> sur les ~20k réels) avec le **pic de 3 000 quasi simultané** — le volume est modeste, le vrai enjeu
> reste le **débit (RPS)** et la **délivrabilité**, pas le coût. Prix indicatifs à revérifier sur les
> grilles fournisseurs (voir dates de consultation en bas de doc).

---

## 3 bis. Absorption du pic (3 000 rafale / 20 000 sur 2 j) par opérateur

> Point clé : ce n'est pas « est-ce que ça passe en volume » (oui pour tous) mais **à quel débit
> et sans throttling / sans blocage anti-abus sur la rafale**.

| Opérateur        | Comportement attendu sur 3 000 en rafale                                                        | Point de vigilance                                                                      |
| ---------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Brevo**        | 🟢 **1 000 RPS confirmé** (doc API) → pic vidé en ~3 s ; IP mutualisée chaude absorbe la rafale | Goulot = **délivrabilité** (réputation IP), plus le rate-limit API                      |
| **Scaleway TEM** | Produit jeune → **quota d'envoi + rate/s à confirmer** face au pic                              | **Demander explicitement** le quota journalier et le rate/s ; tester avant de s'engager |
| **Amazon SES**   | Débit élevé une fois hors sandbox                                                               | **Sortie de sandbox** (200/j au départ) + **hausse de quota** à demander à l'avance     |
| **Mailjet**      | IP mutualisée OK ; rate/s selon plan                                                            | Vérifier rate-limit du plan                                                             |
| **Postmark**     | Transac natif, gros débit                                                                       | US (souveraineté)                                                                       |

**Stratégie d'absorption côté Attendee (indépendante de l'ESP) :**

- La **file BullMQ** (déjà en place) **découple** l'inscription de l'envoi → le pic de 3 000
  **n'attend jamais** l'email ; les mails se drainent au **débit max de l'ESP**.
- On **plafonne le débit worker** au rate-limit annoncé par l'ESP (éviter d'auto-déclencher un
  throttling / blocage anti-spam).
- Le **billet reste accessible immédiatement** via **lien R2 signé** même si l'email part avec
  quelques minutes de décalage → un léger retard d'email **n'est pas un incident bloquant**.
- **Cible opérationnelle :** vider les 3 000 emails du pic en **≤ 5 min** (≈ 10-20 mails/s) →
  à confronter au rate-limit réel de l'ESP retenu.

---

## 4. Focus candidats principaux

### 4.1 Brevo (ex-Sendinblue) — 🇫🇷

- **+** Acteur **français** établi, données UE, RGPD, interface complète, **IP dédiée** en option,
  bonne doc, SMTP + API, logs et webhooks. Écosystème mature.
- **−** Plateforme **mixte marketing + transac** → veiller à ne pas mélanger les réputations ;
  paliers tarifaires qui montent vite si on prend les options (IP dédiée, volume marketing).
- **Verdict :** **valeur sûre**, faible risque, bonne carte « souveraineté FR » à présenter au MEAE.

#### 💶 Les 2 modèles de facturation Brevo (et leurs limites)

> ⚠️ Le **volume est partagé** campagnes marketing **+** transactionnel (même compteur).
> Le **débit est 1 000 RPS** dans les deux modèles (doc API, indépendant du prix).

|                        | **(A) Abonnement mensuel** (Starter+)                               | **(B) Crédits prépayés — pay-as-you-go**                                                                                                                            |
| ---------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Principe**           | Volume mensuel choisi au curseur (5k”100k+)                         | Pack de crédits acheté à l'unité (add-on)                                                                                                                           |
| **Limite journalière** | 🟢 **Aucune** (« Starter includes no daily sending limit »)         | 🟢 **Aucune** (« No daily sending limit »)                                                                                                                          |
| **Expiration**         | 🔴 **Les emails non utilisés expirent chaque mois** (pas de report) | 🟢 **Sans expiration** (« no expiration date »)                                                                                                                     |
| **Prix**               | Palier 30k ≈ ~€30/mois (à confirmer au curseur)                     | Prix/1000 **dégressif au volume**, visible **au curseur in-app** (`app.brevo.com/billing`) — généralement **+ cher/email** que l'abonnement, prix de la flexibilité |
| **Idéal pour**         | Envoi récurrent mensuel                                             | **Événement ponctuel** (LFD) : on achète, on consomme, le reste ne périme pas                                                                                       |

> 🟢 **Seul modèle avec limite journalière = le plan Free (300/j)**. Dès Starter et sur les crédits
> prépayés, **aucune limite/jour** → le pic de 3 000 passe sans plafond journalier.
>
> ⚠️ **Seule vigilance « limite » réelle :** un **compte neuf** peut être soumis à une **validation
> anti-abus** Brevo (brédage temporaire tant que le compte n'est pas vérifié) → **créer et faire
> valider le compte en avance**, ne pas découvrir ça la veille de l'événement.
>
> 💡 **Prix exact des crédits PAYG :** non affiché sur les pages publiques (tableau JS dynamique),
> à relever directement dans **My plan → Manage email credits** dans l'app, ou auprès du support.

### 4.2 Scaleway Transactional Email (TEM) — 🇫🇷 souverain

- **+** **Cloud souverain français** (argument fort MEAE), **100 % transactionnel** (pas de mélange
  marketing), **tarif très bas**, SMTP + API, intégré à l'écosystème Scaleway (si on migre l'infra).
  Le plus aligné sur la contrainte de souveraineté.
- **−** Produit **plus jeune** → réputation IP mutualisée à valider, moins d'options avancées
  (IP dédiée, warm-up assisté) que Brevo/SES ; quotas d'envoi à vérifier vs le pic ; support.
- **Verdict :** **meilleur sur la souveraineté**, à **valider par un test de délivrabilité réel**
  (envoi test vers Gmail/Outlook/Orange/free + inbox placement) avant de s'engager.

### 4.3 Amazon SES — 🇺🇸 (régions UE Paris/Francfort)

- **+** **Le moins cher**, délivrabilité excellente, très scalable, région **UE (Paris)** disponible.
- **−** **Setup plus technique** (IAM, vérif domaine, sortie de sandbox, SNS pour bounces),
  hébergeur **US** (moins vendeur pour un discours de souveraineté MEAE, même en région UE).
- **Verdict :** techniquement excellent et économique, mais **positionnement souveraineté plus faible**.

### 4.4 Autres à connaître

- **Mailjet (Sinch)** — origine FR, UE dispo, simple, alternative crédible à Brevo.
- **Postmark** — **référence délivrabilité transactionnelle**, streams séparés, mais **US** →
  disqualifiant sur le critère souveraineté MEAE.
- **Mailgun (Sinch)** — région UE dispo, orienté développeurs ; correct mais pas d'avantage
  souveraineté décisif.

---

## 5. Recommandation (⚠️ SUPERSEDED — décision finale = **Mailgun**, voir §0)

> 📌La reco initiale ci-dessous (Scaleway/Brevo) date d'avant l'analyse Mailgun du rapport PDF.
> **Choix final acté : Mailgun EU / IP partagée** (§0).

| Scénario                                    | Choix reco                                                                         |
| ------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Priorité souveraineté MEAE (le cas ici)** | 🥇 **Scaleway TEM** — _sous réserve du test de délivrabilité_ → sinon 🥈 **Brevo** |
| **Priorité sécurité/robustesse éprouvée**   | 🥇 **Brevo** (valeur sûre FR)                                                      |
| **Priorité coût/scalabilité pure**          | Amazon SES (région Paris) — mais souveraineté moins vendeuse                       |

**Proposition :** retenir **Scaleway TEM en cible** (meilleur discours souveraineté pour le MEAE),
avec **Brevo en plan B** si le test de délivrabilité Scaleway n'est pas concluant. Les deux
s'intègrent en **relais SMTP nodemailer** (effort mini). Trancher **après un test d'inbox placement**
sur les 4 grands FAI FR (Gmail, Outlook/Hotmail, Orange, Free).

---

## 6. Points d'intégration & warm-up (quel que soit l'ESP)

- **Migration technique :** basculer `nodemailer` du SMTP direct vers le **relais SMTP de l'ESP**
  (host/port/user/pass en variables d'env). API optionnelle dans un 2e temps.
- **Authentification domaine (à faire J1) :** configurer **SPF + DKIM + DMARC** sur le domaine
  expéditeur → **délai de propagation DNS**, à lancer tout de suite.
- **Warm-up :** si **IP dédiée**, monter le volume **progressivement sur 1-2 semaines** avant le pic
  (sinon risque de spam/blocage). Si IP mutualisée, warm-up géré par l'ESP mais **valider le quota
  d'envoi** face au pic. → **c'est le délai calendaire critique du chantier C.**
- **Dimensionnement rafale (3 000 / 20 000 sur 2 j) :** **confirmer par écrit** auprès de l'ESP le
  **rate-limit (emails/s)** et le **quota journalier** ; **plafonner le worker BullMQ** à ce rate
  pour ne pas déclencher de throttling ; **tester une rafale** (ex. 500-1000 emails) en staging.
- **Séquencement :** email envoyé **après** génération du billet PDF (déjà acté au diagnostic §3).
- **Observabilité :** brancher les **webhooks bounces/plaintes** pour suivre les échecs pendant l'event.

---

## 7. Décision (à remplir)

- [x] ESP retenu : **Mailgun** (région EU, IP partagée) — **décidé 11/07/2026** (sous-domaine `mail.attendee.fr`)
- [ ] Test d'inbox placement réalisé (Gmail/Outlook/Orange/Free) → résultat **\_**
- [ ] IP dédiée ou mutualisée → **\_**
- [ ] **Rate-limit (emails/s) confirmé par l'ESP** → **\_**
- [ ] **Quota journalier confirmé ≥ ~15 000/jour** (pour ~20k sur 2 j) → **\_**
- [ ] **Test rafale réalisé** (500-1000 emails en staging, sans throttling) → **\_**
- [ ] Worker BullMQ plafonné au rate ESP → **\_**
- [ ] SPF/DKIM/DMARC configurés (date) → **\_**
- [ ] Warm-up lancé (date début) → **\_**
- [ ] Quota d'envoi confirmé ≥ pic attendu → **\_**

---

## 8. Sources (données vérifiées, consultées le 1er juillet 2026)

> ⚠️ Les grilles tarifaires et quotas changent souvent — revérifier avant décision.

| ESP              | Rate-limit (RPS)                                                                                                 | Tarif marginal                                                                                                                                                                                                                                                                             | Source                                                                                                                                                               |
| ---------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Brevo**        | **1 000 RPS** (`POST /v3/smtp/email`, General — tous plans) ; 2 000 (Advanced/Pro) ; 6 000 (Extended/Enterprise) | Volume **campagnes + transac partagé**. **(A) Abonnement** Starter+ : **aucune limite/jour**, volume **expire chaque mois** · **(B) Crédits prépayés** : **aucune limite/jour, sans expiration** (prix au curseur in-app). ⚠️ **Free = 300/j** ; compte neuf soumis à validation anti-abus | [api-limits](https://developers.brevo.com/docs/api-limits) · [pricing](https://www.brevo.com/pricing/) · [plans](https://help.brevo.com/hc/en-us/articles/208589409) |
| **Scaleway TEM** | Non publié (quota compte à demander au support)                                                                  | Essential **PAYG** : 300 gratuits, puis **+€0,25/1000** · Scale **€80/mois** : 100k inclus + IP dédiée managée + SLA 99,9 %, puis **+€0,20/1000**                                                                                                                                          | [TEM](https://www.scaleway.com/en/transactional-email-tem/)                                                                                                          |
| **Amazon SES**   | Sandbox : **1 email/s, 200/24 h** ; hors sandbox : ajustable (élevé)                                             | **~0,10 $/1000** (facturation au destinataire) ; pièces jointes jusqu'à 40 Mo (SMTP/v2)                                                                                                                                                                                                    | [quotas SES](https://docs.aws.amazon.com/ses/latest/dg/quotas.html)                                                                                                  |
| **Mailjet**      | Transac « high rate limit », **non chiffré** ; 429 si dépassement                                                | Free 6k/mois (200/j) · Starter $9 (8k) · Essential $17 (15k) · Premium $27 (15k) · paliers sup. au curseur                                                                                                                                                                                 | [pricing](https://www.mailjet.com/pricing/) · [rate-limits](https://dev.mailjet.com/email/reference/overview/rate-limits/)                                           |
| **Postmark**     | Élevé, **limite journalière illimitée** ; RPS non chiffré                                                        | Basic **$15/mois** (10k, +$1,80/1000) · Pro **$16,50/mois** (10k, +$1,30/1000) · IP dédiée $50/mois (≥300k/mois)                                                                                                                                                                           | [pricing](https://postmarkapp.com/pricing)                                                                                                                           |

**Rappel transversal :** pour ~20-30k emails sur 2 jours avec un pic de 3 000, **le volume et le débit
ne sont jamais le facteur limitant** (même le plan gratuit Brevo tient 1 000 RPS). Le seul vrai enjeu
reste la **délivrabilité** (IP mutualisée chaude vs dédiée à warm-up, SPF/DKIM/DMARC) et, pour SES, la
**sortie de sandbox** à anticiper. La **file BullMQ** côté Attendee lisse le pic quel que soit l'ESP.
