# 06 — Points de vigilance vs cahier des charges MEAE / La Fabrique de la Diplomatie

> **À QUOI SERT CE FICHIER :** confrontation du [cahier des charges officiel](../../../..) (MEAE —
> événement diplomatique gratuit, 4-5 septembre 2026, Université Sorbonne Nouvelle, ~20 000
> participants sur 2 jours) avec l'état réel du code et des chantiers LFD 2026 (voir
> [03-suivi-chantiers.md](./03-suivi-chantiers.md)). Le total de jauge du cahier des charges
> (12 360 inscriptions, §2.2) correspond à la base warm-up "~12 400 emails" citée en C3 : il s'agit
> bien du même événement.
>
> **Méthode :** chaque point a été vérifié directement dans le code (back/front/mobile) — pas
> seulement sur la base du suivi de chantiers — conformément au principe "code = source de vérité
> pour les faits actuels".
>
> **Dernière mise à jour :** 2026-07-18

---

## Ce qui est déjà solide (vérifié dans le code)

| Exigence cahier des charges                             | Statut                | Preuve dans le code                                                                                                         |
| ------------------------------------------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| §7 Back-office multi-rôles (admin/opérateur/lecteur)    | ✅ Dépasse l'exigence | `SUPER_ADMIN/ADMIN/MANAGER/VIEWER/PARTNER/HOSTESS` — `attendee-ems-back/src/modules/roles/roles.service.ts`                 |
| §5.3 Consentement RGPD sur formulaire public            | ✅ Fait               | Case à cocher obligatoire + lien politique de confidentialité — `attendee-ems-front/src/pages/PublicRegistration/index.tsx` |
| §4.2 / §6 / §7 Export CSV/Excel + QR codes              | ✅ Fait               | `ExportColumnsModal.tsx` (colonnes personnalisées + option QR codes)                                                        |
| §4.1 Mode offline scan mobile (dégradé + sync différée) | ✅ Fait (mature)      | Queue de replay, résolution de conflits, NetInfo — `attendee-ems-mobile/context/constraints/mobile-offline.md` + code       |
| §6.2 Journalisation des accès/actions back-office       | ✅ Fait               | `setAudit(...)` branché sur les inscriptions publiques (`public.controller.ts`)                                             |
| Anti-doublon inscription (event/attendee)               | ✅ Fait               | Vérification unicité dans `public.service.ts`                                                                               |

---

## Points de vigilance réels (gaps ou risques)

### 🔴 1. CAPTCHA / anti-bot (§6.2) — absent

Le seul garde-fou trouvé est un `@Throttle({ default: { limit: 100, ttl: 60_000 } })` sur les endpoints
d'inscription publique (`public.controller.ts`), **volontairement laissé haut** pour ne pas bloquer les
inscriptions légitimes derrière un NAT d'entreprise/ministère. Le cahier des charges exige explicitement
une "protection contre les attaques de type bot, scraping massif et tentatives d'inscription automatisée
(CAPTCHA ou équivalent)" — exactement le scénario "jauge qui se remplit en quelques minutes à l'ouverture
d'un créneau" décrit en §2.3. **Aucun chantier du suivi ne couvre ce point aujourd'hui.**

### 🔴 2. Chantier J — portier Redis / 3000 requêtes simultanées (§6.1) — 35 %

C'est la seule exigence **chiffrée** du cahier des charges ("capacité à absorber des pics de connexions
simultanées... minimum 3000 requêtes simultanées"). C'est aussi, après H, le chantier le moins avancé.
Reste : portier Redis/compteur atomique, cache public léger, Socket.IO Redis adapter, **et surtout un load
test de validation réel** — actuellement aucun n'a été mené à cette échelle.

### 🔴 3. Chantier F (HA) reporté post-event vs SLA 99,9 % exigé (§6.1)

Le cahier des charges demande un **SLA de disponibilité de 99,9 % minimum pendant toute la durée de
l'événement (J-15 à J+1)**. Le chantier F (haute disponibilité / réplication) est explicitement **reporté
à la migration GCP post-event** dans le suivi. Le monitoring (0-MON, fait à 100 %) donne de la visibilité
sur les incidents mais **ne compense pas l'absence de redondance** : un incident sur le serveur unique
reste un point de rupture direct du SLA contractuel. À signaler explicitement à qui pilote la relation
MEAE / La Fabrique de la Diplomatie — c'est une contradiction entre une décision de descope interne et un
engagement client potentiel.

### 🟠 4. Chantier C2.1 — billet PDF + email (§3.3) — 20 %

Le billet numérique nominatif avec QR code est une **exigence fonctionnelle de premier plan** (§3.3), pas
un nice-to-have. Le pipeline (`ticket.generate` → PDF → `email.send`) n'est cadré que depuis le 17/07 et
reste à ~20 % avec ~5-7 j-dev restants estimés. Risque calendaire si le reste à faire glisse.

### 🟠 5. "+1 aux activités" (§3.1) — non retrouvé dans le code

Le cahier des charges autorise explicitement "s'inscrire avec un +1 aux deux activités". Aucune trace
d'un flux d'inscription accompagnant/plus-one n'a été trouvée dans le code exploré (`PublicRegistration`,
`public.service.ts`). À confirmer explicitement avec les owners de BIL/H : soit c'est prévu et pas encore
vu, soit c'est un gap fonctionnel non tracké dans le suivi de chantiers actuel.

### 🟠 6. Plafonds "max 2 activités/jour" et "max 2 inscriptions/jour par appareil+email" (§3.1, §3.2)

La vérification anti-doublon actuelle porte sur _un event donné_ (unicité event/attendee). Aucune règle
explicite trouvée pour :

- limiter à 2 le nombre d'activités choisies par jour et par personne (tous créneaux confondus),
- limiter les inscriptions par adresse IP/appareil (le `@Throttle` actuel est un anti-abus générique, pas
  une règle métier "2 max par jour et par appareil/email").

### 🟠 7. Chantier K — PCA documenté (§6.1) — 10 %

Le cahier des charges exige un "plan de continuité en cas d'incident (PCA) documenté". Le chantier K
(résilience event / checklist / runbooks) n'est qu'à 10 % dans le suivi — checklist créée, vérifications
à dérouler.

### 🟡 8. Volet contractuel RGPD (§5.1) — non porté par un chantier

Rien dans le suivi de chantiers ne couvre explicitement :

- le DPA (Data Processing Agreement) signé avant mise en production,
- la déclaration formelle de localisation des serveurs en UE,
- la politique documentée de conservation et de suppression des données,
- les coordonnées du DPO à fournir.

Ce sont plutôt des livrables légal/ops que dev, mais personne ne semble les porter actuellement dans le
plan d'action LFD 2026.

### 🟡 9. Exception "vendredi matin" — lien d'invitation privé (§2.3)

Le cahier des charges prévoit un mode d'inscription différent pour le créneau du vendredi matin : "lien
d'invitation privé, disponible plusieurs jours voire semaines à l'avance" (par opposition à l'ouverture
J-0 H-2 des autres créneaux). À confirmer que la plateforme supporte bien un mode d'inscription
anticipée/privée distinct du flux public standard (potentiellement via un token dédié non diffusé
publiquement), plutôt que de le découvrir tardivement comme un cas particulier non couvert.

### 🟡 10. Fenêtre d'ouverture des créneaux (§2.1) — logique min/max à vérifier

Règle : ouverture "au moins 30 minutes avant, au maximum 4 heures avant" le début de l'activité. Le
chantier H expose bien une configuration d'ouverture (`registration_opens_at`), mais la contrainte
"min 30 min / max 4h" glissante par créneau n'a pas été vérifiée précisément dans le code — à confirmer
qu'elle est paramétrable ainsi et pas juste une date fixe unique.

---

## Priorisation suggérée (vs capacité restante ~30 j-dev)

1. **CAPTCHA/anti-bot** (point 1) — coût probablement faible (intégration reCAPTCHA/hCaptcha/Turnstile),
   impact élevé face au scénario de pics décrit dans le cahier des charges.
2. **Load test réel de J** (point 2) — condition de validation explicite du cahier des charges (3000 req
   simultanées), à ne pas découvrir en prod le jour J.
3. **Décision explicite sur le SLA/HA** (point 3) — pas forcément du dev, mais une clarification/
   communication à faire rapidement plutôt qu'un silence sur la contradiction.
4. **C2.1 billet PDF** (point 4) — déjà priorisé en J2 dans le suivi, à ne pas laisser glisser.
5. **Vérifier "+1" et plafonds d'inscription** (points 5, 6) — vérification rapide à faible coût, à
   trancher avec les owners produit avant de considérer H/BIL "terminés".
6. **PCA (K)** et **volet RGPD contractuel** (points 7, 8) — à sortir de l'angle mort, même si ce n'est
   pas uniquement du développement.
