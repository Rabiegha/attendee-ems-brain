# Diagnostic — Chaîne Email / Billet PDF / Wallet (LFD 2026)

> **Objet :** aligner la note de cadrage ChoYou (générique) avec **l'état réel du code Attendee**,
> et définir ce qui reste **à construire** pour l'événement La Fabrique de la Diplomatie 2026
> (MEAE, ~12 400 inscriptions, 4–5 sept 2026).
> **Statut :** diagnostic — décisions cadrées, implémentation à planifier.
> **Repo principal :** `attendee-ems-back`.

---

## 0. Avertissement sur la note ChoYou

La note de cadrage ChoYou décrit une stack **générique** (PHP / wkhtmltopdf / sendmail, etc.)
qui **ne correspond pas** à la réalité d'Attendee (NestJS / Puppeteer / nodemailer / R2 / BullMQ).
Elle est utile comme **liste de besoins fonctionnels**, pas comme référence technique.
Ce document remet chaque besoin en face du **code réel**.

---

## 1. Note ChoYou vs État réel vs À construire

| Sujet | Ce que dit la note ChoYou | État réel Attendee (vérifié dans le code) | À construire |
|---|---|---|---|
| **Génération QR** | QR encodant un identifiant | ✅ Existe. QR = **UUID brut** de la registration (`QRCode.toBuffer(registrationId)`), `email.service.ts` L277-291. | 🔸 **Signature HMAC** du payload (voir §5 sécurité). |
| **QR dans l'email** | QR affiché dans l'email | ✅ Existe. Pièce jointe **CID** `cid:qrcode@attendee`, déclenchée si le template contient `{{qrCode}}`. | RAS (fonctionne). |
| **Endpoint QR public** | — | ⚠️ Route **publique non authentifiée** `/qrcode/:registrationId`, seule validation = longueur ≥ 10 (`qrcode.controller.ts`). | 🔸 À durcir (signature + éventuel rate-limit). |
| **Stack email** | sendmail / SMTP | ✅ **nodemailer SMTP direct** (PAS un ESP). | 🔴 **Migration ESP** (Brevo / Scaleway) — délivrabilité ~12 400 envois. Plus gros chantier email. |
| **Envoi asynchrone** | File d'envoi | ✅ Emails de confirmation/admin **mis en file BullMQ après commit** (`public.service.ts`), branche `staging`. | Fusionner `staging` → prod le moment venu. |
| **Génération badge PDF** | wkhtmltopdf | ✅ **Puppeteer + Chromium**, génère **PDF + PNG**, upload **R2** (`badges/{registrationId}/badge.*`), `badge-generation.service.ts` L780+. | RAS sur la génération. |
| **Idempotence badge** | — | ✅ Early-return : si badge `completed` existe → renvoyé en cache (pas de régénération). | Réutiliser ce **pattern d'idempotence** pour wallet. |
| **PDF dans l'email** | Email contient le PDF | ❌ **Aujourd'hui le PDF n'est PAS joint à l'email** (généré séparément, au moment de l'impression). | 🔴 **Joindre le PDF généré** à l'email (+ lien de secours). |
| **Wallet Apple** | Pass Apple Wallet | ❌ **N'existe pas.** | ⚪ **V2** — `.pkpass` signé (certif Apple). Hors périmètre event. |
| **Wallet Google** | Pass Google Wallet | ❌ **N'existe pas.** | ⚪ **V2** — Google Wallet API (compte service). Hors périmètre event. |
| **Anti-pic inscription** | Waiting Room / CDN / Turnstile | ❌ Pas en place. | 🔴 Infra Cloudflare (Waiting Room + Turnstile) — voir plan de continuité / scaling. |
| **Jauge atomique** | Compteur Redis (DECR) | ⚠️ Capacité gérée par **COUNT en transaction DB** (= plafond de sérialisation mesuré). | 🔸 Compteur atomique **Redis (DECR)** — levier scaling (cf. workstream scaling). |

Légende : ✅ existe / ⚠️ existe mais à durcir / ❌ absent / 🔴 chantier majeur / 🔸 amélioration ciblée / ⚪ reporté V2.

---

## 2. Plan par artefact

### 2.1 QR code
- **État :** opérationnel (UUID brut, CID dans l'email + route publique).
- **Cible LFD :**
  1. **Signer le payload** (HMAC) au lieu de l'UUID nu → un QR ne peut plus être forgé/deviné.
  2. Le scan (mobile / print-client) valide la **signature** avant de résoudre la registration.
  3. Durcir / éventuellement supprimer la route publique `/qrcode/:id` au profit du QR signé embarqué.
- **Effort :** faible-moyen (modif encodage + validation au scan).

### 2.2 Billet PDF
- **État :** génération Puppeteer→R2 OK + idempotence. Mais **pas joint à l'email**.
- **Cible LFD :**
  1. À la confirmation, **générer (ou réutiliser le cache R2)** le PDF du billet.
  2. **Joindre le PDF** à l'email + **lien de secours** (URL R2 signée) en cas de pièce jointe bloquée.
  3. **Envoyer l'email seulement quand le billet est prêt** (séquencement : billet → email).
- **Effort :** moyen (orchestration file : générer → attacher → envoyer).

### 2.3 Wallet (Apple + Google) — **nice-to-have V2, HORS périmètre event**
- **État :** inexistant.
- **Décision :** **reporté en V2.** Le wallet n'est **pas** requis pour LFD 2026 (QR + PDF suffisent
  pour l'event). Documenté ici pour la cible, mais **ne bloque pas** la livraison de l'événement.
- **Cible V2 :** produire un **pass uniforme** à côté du QR + PDF, **dans la même file**, même
  clé d'idempotence `wallet:<registrationId>:<platform>`, même stockage **R2**.
  - **Apple** : générer un `.pkpass` **signé** (certificat Pass Type ID Apple).
  - **Google** : créer l'objet via **Google Wallet API** (compte de service Google Cloud).
  - Les deux fichiers/liens seraient **joints/inclus dans l'email**.
- **Effort :** élevé — **dépend de prérequis comptes/certifs** (voir §4). **À traiter en V2**, pas avant l'event.

> **🧩 Décision 2026-07-08 — découper G en 4, ne PAS tout mettre dans B event-critique.**
> Question posée : « faire Apple + Google **en même temps** que le PDF, c'est à peu près la même
> chose (même génération sur Cloud Run) ? » → **vrai pour le harnais, faux pour la génération par format.**
>
> **Ce qui est commun (le harnais) :** déclencheur post-inscription, source de données, **même token
> QR de check-in**, **service de génération Cloud Run** (stateless), livraison email. Le check-in
> est **découplé du format** (le scanner lit le même token) → ça vaut le coup de bâtir le harnais **une fois**.
>
> **Ce qui n'est PAS la même chose (le piège) :**
> - **PDF** = rendu pur, **aucune** dépendance externe, couvre **100 %** des gens → le must-have.
> - **Google Wallet** = JSON (`EventTicketObject`) + **JWT signé** ; une fois l'**Issuer** approuvé
>   → **léger**. Le wallet « facile ».
> - **Apple Wallet** = bundle `.pkpass` **signé** (PKCS#7) : **compte Apple Developer** (99 $/an) +
>   **Pass Type ID cert** + WWDR + assets @2x/@3x + **renouvellement annuel du cert** + secrets serveur.
>   → **délai externe** (création compte/cert) + nouveaux **modes de panne op** → **hors chemin critique**.
>
> **Reco actée (découpe de G) :**
> 1. **G-harnais → fusionné dans B maintenant** : interface de génération **agnostique** `generate(format, data)`,
>    PDF = 1ʳᵉ implémentation. Coût du harnais payé une fois, zéro refacto ensuite.
> 2. **G-onboarding → démarrer aujourd'hui, en parallèle** (0 dev, pur délai admin) : ouvrir le **compte
>    Apple Developer + Pass Type ID cert** ET l'**Issuer Google**. Dégage le délai externe du chemin critique.
> 3. **G-Google → fast-follow après le PDF** si buffer (JSON + JWT une fois l'Issuer prêt).
> 4. **G-Apple → après PDF solide**, avant l'event si buffer, mais **hors chemin critique** ; le PDF reste le fallback 100 %.
>
> **Coût caché du « en même temps » :** faire les 3 d'un coup **triple la surface QA + load-test**
> simultanément sur un service **sur le chemin critique de l'email** → livrer le PDF, prouver le pipeline
> sous charge, PUIS empiler les wallets qui réutilisent le harnais éprouvé.

---

## 3. Décisions actées

1. L'email contiendra **le billet PDF généré** (et **le fichier wallet en V2** seulement).
2. **PDF joint** + **lien de secours** (URL R2 signée).
3. L'email est **envoyé après que le billet est prêt** (séquencement strict).
4. Le wallet (V2) réutilisera **le pattern de la file badge** (idempotence + R2), pas une voie parallèle.
5. La note ChoYou est traitée comme **cahier des besoins**, pas comme spec technique.
6. **Wallet = nice-to-have V2**, hors périmètre de l'event LFD 2026.

---

## 4. Comptes & prérequis wallet (réponse Q6/Q7) — V2 uniquement

> **Propriété des comptes :** détenus par **toi** (Rabiegha) ; **tu feras les demandes**.
> **État :** **tout est à créer** — mais **seulement quand la V2 wallet démarrera** (pas pour l'event).

| Prérequis | Pour quoi | Délai typique | Action |
|---|---|---|---|
| **Apple Developer Program** (compte payant) | Émettre le certificat **Pass Type ID** (signer les `.pkpass`) | Inscription + validation (peut prendre plusieurs jours) | ⚪ V2 |
| **Certificat Pass Type ID** (Apple) | Signature des passes Apple Wallet | Après compte actif | ⚪ V2 |
| **Projet Google Cloud + Google Wallet API** | Émettre les passes Google Wallet | Activation API + compte de service | ⚪ V2 |
| **Compte de service Google (clé JSON)** | Authentifier l'émission des passes côté serveur | Rapide une fois le projet créé | ⚪ V2 |

> ℹ️ Ces prérequis ne sont **pas** sur le chemin critique de l'event (wallet reporté en V2).
> À ouvrir **au lancement de la V2 wallet**, en gardant en tête le délai de validation Apple.

---

## 5. Sécurité — point d'attention QR

Aujourd'hui le QR encode l'**UUID brut** de la registration, et la route `/qrcode/:registrationId`
est **publique et non authentifiée** (seule garde : longueur ≥ 10). Conséquences :

- Un UUID connu/deviné permet de **régénérer le QR** correspondant.
- Le QR n'est **pas vérifiable** : le scanner fait confiance à l'identifiant tel quel.

**Reco :** signer le contenu du QR (**HMAC** avec secret serveur), valider la signature au scan,
et restreindre/supprimer la route publique. À cadrer avant l'event (données nominatives MEAE).

---

## 6. Questions ouvertes restantes

1. **Format du billet PDF** : modèle visuel validé par le client MEAE ? (sinon, qui le fournit ?)
2. **ESP retenu** : Brevo ou Scaleway ? (impacte délivrabilité + intégration nodemailer). → **À trancher.**
3. **Volumétrie d'envoi** : pic d'emails attendu (tous d'un coup à l'ouverture, ou étalé ?).
4. ~~Wallet obligatoire pour l'event ?~~ → **Tranché : nice-to-have V2**, hors périmètre event.
5. **Rétention des fichiers** PDF (et wallet en V2) sur R2 (durée + purge post-event).

---

## 7. Synthèse priorisée

| Priorité | Chantier | Pourquoi |
|---|---|---|
| 🔴 P1 | Migration ESP (délivrabilité ~12 400 envois) — **choix Brevo/Scaleway à trancher** | Risque délivrabilité = risque client |
| 🔴 P2 | Joindre PDF à l'email + lien de secours + séquencement | Décision actée, base du billet |
| 🔸 P2 | Signer le QR (HMAC) + durcir route publique | Sécurité données MEAE |
| 🔸 P3 | Jauge atomique Redis + infra Cloudflare anti-pic | Cf. workstream scaling |
| ⚪ V2 | Wallet Apple + Google (file + R2 + idempotence) | **Nice-to-have, hors event** — après prérequis comptes/certifs |
