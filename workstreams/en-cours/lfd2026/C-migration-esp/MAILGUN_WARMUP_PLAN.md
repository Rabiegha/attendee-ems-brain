# MAILGUN — Plan de warm-up domaine & mise en production (Chantier C1)

> **Objet :** dérouler, de la création du sous-domaine jusqu'au pic LFD 2026, la préparation
> **Mailgun (région EU, IP partagée)** et le **warm-up du domaine d'envoi**.
> **Décision ESP :** [esp-choix-mailgun.md](./esp-choix-mailgun.md) · **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-C1)
> **Sous-domaine d'envoi retenu :** `mail.attendee.fr` · **Région :** EU (`api.eu.mailgun.net`).
>
> ⚠️ **Dépendance bloquante : accès OVH (DNS).** Sans accès à la zone DNS OVH d'`attendee.fr`,
> impossible de créer le sous-domaine, publier SPF/DKIM/DMARC, vérifier le domaine et **démarrer
> le warm-up**. → **C1 est bloqué tant que l'accès OVH n'est pas obtenu.** (C2 avançable en parallèle.)

---

## ⏱️ Durées de référence (IP partagée)

| Question                                  | Réponse retenue                                                   |
| ----------------------------------------- | ----------------------------------------------------------------- |
| Durée **minimale** avant un gros envoi    | **~2 semaines** de vie du domaine                                 |
| Durée **idéale**                          | **3 à 4 semaines**                                                |
| « Vie » du domaine avant bonne réputation | ≥ 2 sem d'envois **réels et engagés** (faible bounce, ouvertures) |

> Même en **IP partagée** (pool Mailgun déjà chaud), on **chauffe le domaine/sous-domaine** :
> c'est la **réputation du domaine expéditeur** qu'on construit, pas celle de l'IP.
> ➡️ **Rétro-planning : lancer C1 au plus tard ~3-4 semaines avant le pic d'inscriptions.**

---

## Phase 1 — Prérequis (setup Mailgun + DNS)

> ⛔ **Nécessite l'accès OVH.**

- [ ] **Acheter le plan Mailgun** retenu (Basic ~$33/mois, ou Foundation ~$35 pour + de rétention logs)
- [ ] **Ajouter le domaine** d'envoi dans Mailgun → **choisir la région EU** (message data region-bound EU)
- [ ] Créer le **sous-domaine d'envoi** `mail.attendee.fr` (séparé du domaine principal pour isoler la réputation)
- [ ] **Générer les clés API** (région EU) + clé SMTP si transport nodemailer
- [ ] Publier les DNS **chez OVH** :
  - [ ] **SPF** (TXT)
  - [ ] **DKIM** (clé fournie par Mailgun)
  - [ ] **DMARC** (politique minimale au départ : `p=none`, puis durcir)
  - [ ] **CNAME de tracking** + **MX / Return-Path** si demandés par Mailgun
- [ ] **Vérifier la propagation DNS** (Mailgun annonce **24-48 h**)
- [ ] Domaine **« Verified »** dans le dashboard Mailgun

---

## Phase 2 — Warm-up (montée en charge progressive)

> Démarrer avec de **vrais emails transactionnels** à audience **engagée** (pas des envois à vide).
> Planning indicatif à ajuster selon les métriques (bounce/spam).

| Jour | Volume cible                                                               |
| ---- | -------------------------------------------------------------------------- |
| J1   | 20 – 50 emails                                                             |
| J2   | ~100                                                                       |
| J3   | ~250                                                                       |
| J4   | ~500                                                                       |
| J5   | ~1 000                                                                     |
| J6   | ~2 000                                                                     |
| J7+  | montée progressive jusqu'au volume du pic (~3 000 en rafale, ~10-15k/jour) |

**Règles pendant le warm-up :**

- Surveiller **bounce rate** et **plaintes spam** à chaque palier ; **ralentir** si ça grimpe.
- Envoyer via la **queue BullMQ** en plafonnant le débit worker (ne pas déclencher d'anti-abus).
- Garder le **lien R2 signé** du billet comme secours si un email tarde.

---

## Phase 3 — Validation (avant l'event)

**Authentification & config**

- [ ] SPF OK
- [ ] DKIM OK
- [ ] DMARC OK
- [ ] Domaine **vérifié**

**Délivrabilité (inbox placement)**

- [ ] Bounce OK (soft/hard gérés)
- [ ] Delivery OK
- [ ] **Gmail** (inbox, pas spam)
- [ ] **Outlook / Hotmail**
- [ ] **Yahoo**

**Exploitation**

- [ ] **Webhooks** fonctionnels (`delivered`, `bounce`, `spam`, `blocked`)
- [ ] **Monitoring** opérationnel (taux de livraison, file BullMQ, alertes)

---

## Rétro-planning type (à caler sur la date du pic)

| Fenêtre         | Actions                                                   |
| --------------- | --------------------------------------------------------- |
| **J-14 à J-10** | Créer domaine/sous-domaine · publier SPF/DKIM/DMARC (OVH) |
| **J-10 à J-7**  | Vérifier DNS · envoyer quelques emails internes           |
| **J-7 à J-5**   | Envoyer 100 → 500 emails transactionnels réels            |
| **J-4 à J-2**   | Monter progressivement à 1k puis 2k/jour                  |
| **J-1**         | Test à blanc · webhooks · monitoring · alertes            |
| **J / J+1**     | Pic réel via queue + backoff                              |
