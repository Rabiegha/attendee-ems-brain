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
> **Dernière mise à jour :** 2026-07-19

---

## Ce qui est déjà solide (vérifié dans le code)

| Exigence cahier des charges                             | Statut                | Preuve dans le code                                                                                                         |
| ------------------------------------------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| §7 Moteur RBAC Attendee                                 | 🟡 Socle disponible  | Attendee possède rôles/permissions/scopes ; intégration BIL non faite                                                     |
| §5.3 Consentement RGPD sur formulaire public            | ✅ Fait               | Case à cocher obligatoire + lien politique de confidentialité — `attendee-ems-front/src/pages/PublicRegistration/index.tsx` |
| §4.2 / §6 / §7 Export CSV/Excel + QR codes              | ✅ Fait               | `ExportColumnsModal.tsx` (colonnes personnalisées + option QR codes)                                                        |
| §4.1 Mode offline scan mobile (dégradé + sync différée) | 🟠 Socle partiel     | Queue/replay et cache session présents ; doublons session, cold start et recette réelle non prouvés                         |
| §6.2 Socle de journalisation                            | 🟡 Socle disponible  | `audit_logs`, intercepteur global et viewer root existent ; couverture BIL portée par O                                     |
| Anti-doublon inscription (event/attendee)               | ✅ Fait               | Vérification unicité dans `public.service.ts`                                                                               |

---

## Points de vigilance réels (gaps ou risques)

### 🔴 0. Back-office BIL non RBAC (§7) — gap confirmé

Le CDC exige un accès multi-utilisateurs simultané avec droits administrateur, opérateur et lecteur.
Attendee possède un moteur RBAC, mais `back-office-event` ne l'applique pas : tout login Attendee
réussi pose une session admin binaire et `index.php`/`save.php` ne contrôlent aucun rôle. Le chantier
BIL porte désormais la création des trois rôles LFD, les guards serveur par action, l'UI adaptée et
les tests allow/deny. Le MEAE pourra attribuer un rôle prédéfini ; l'édition libre de la matrice de
permissions reste hors MVP car elle n'est pas explicitement demandée par le CDC.

### 🔴 0 bis. Journalisation des accès/actions BIL (§6.2) — couverture partielle

L'audit statique du 19/07 confirme un socle réutilisable dans Attendee : table PostgreSQL
`audit_logs`, intercepteur global, métadonnées JSON et viewer super-admin. Il ne confirme pas la
conformité du parcours BIL :

- succès de login enrichi, mais IP/user-agent réels perdus derrière le proxy PHP ;
- échec de login écrit sous forme générique, sans email tenté ni raison exploitable ;
- logout PHP non propagé à Attendee ;
- modification d'inscription et changement de rôle sans avant/après suffisant ;
- changement de jauge visible comme `session.update`, sans ancienne/nouvelle valeur ;
- bouton ouvrir/fermer BIL fondé sur `manual_state` local alors que `save.php` réactive le token
  public Attendee : l'audit ne représente pas fidèlement l'action de l'opérateur ;
- exports avancés Attendee bien enrichis, mais non encore intégrés dans BIL ;
- aucun E2E ne vérifie actuellement les lignes réellement écrites dans `audit_logs`.

Le chantier [O — audit métier et sécurité LFD](./O-audit-lfd/README.md) devient propriétaire du
contrat `AuditEventV1`, de la fiabilité, du stockage, des E2E et de la preuve CDC. `O-BIL` est son lot
d'intégration avec Corentin ; aucun sous-chantier `BIL-AUDIT` séparé ne doit être créé ou compté.
La plateforme générale [P](./P-plateforme-journalisation/README.md) reste entièrement post-event.

### 🔴 1. CAPTCHA / anti-bot (§6.2) — absent

Le seul garde-fou trouvé est un `@Throttle({ default: { limit: 100, ttl: 60_000 } })` sur les endpoints
d'inscription publique (`public.controller.ts`), **volontairement laissé haut** pour ne pas bloquer les
inscriptions légitimes derrière un NAT d'entreprise/ministère. Le cahier des charges exige explicitement
une "protection contre les attaques de type bot, scraping massif et tentatives d'inscription automatisée
(CAPTCHA ou équivalent)" — exactement le scénario "jauge qui se remplit en quelques minutes à l'ouverture
d'un créneau" décrit en §2.3. Le sous-chantier [N-ANTI](./N-architecture-event-ready/N-ANTI-protection-anti-bot.md)
couvre désormais ce gap. Risque supplémentaire confirmé : le proxy PHP appelle Attendee depuis une IP
serveur unique ; le quota actuel pourrait agréger tous les visiteurs si l'identité réseau n'est pas corrigée.

### 🔴 2. Chantier J — portier Redis / 3000 requêtes simultanées (§6.1) — 35 %

C'est la seule exigence **chiffrée** du cahier des charges ("capacité à absorber des pics de connexions
simultanées... minimum 3000 requêtes simultanées"). C'est aussi, après H, le chantier le moins avancé.
Reste : portier Redis/compteur atomique, cache public léger, Socket.IO Redis adapter, **et surtout un load
test de validation réel** — actuellement aucun n'a été mené à cette échelle.

### 🔴 2 bis. Tableau de bord des entrées et export de présence — couverture partielle

L'audit code du 19/07 confirme que `registered_count` mesure les choix de session `confirmed`, pas les
personnes réellement entrées. Attendee possède déjà deux calculs utiles sur les scans au statut
`admitted` : présence courante (dernier scan IN/OUT) et visiteurs entrés au moins une fois. Le front
les affiche partiellement dans l'onglet Sessions.

Ce socle ne suffit pas encore au CDC :

- après un scan mobile, aucun événement de présence n'est émis ; l'écran ouvert sur un autre appareil
  ne se rafraîchit donc pas réellement en temps réel ;
- il n'existe pas de dashboard global salle + créneau ; les stats se chargent à l'ouverture de chaque
  session ;
- la barre existante divise la présence par `session.capacity`, alors que le remplissage physique doit
  utiliser `checkin_capacity ?? capacity` ;
- `export-sessions` contient les emails et heures d'entrée, mais la feuille d'une session part des
  personnes ayant déjà un scan : les confirmés jamais scannés, donc absents, en sont exclus ;
- aucun statut explicite `Présent`/`Absent` n'est produit. Un participant entré puis sorti doit rester
  `Présent` dans le bilan post-event, même s'il n'est plus présent dans la salle.

Le lot [J-ENTREES](./J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd) couvre le
snapshot et le rafraîchissement live ; H/L9.1 porte la vérité atomique ;
[BIL-ENTREES](./BIL-billetterie/README.md#bil-entrees--vue-opérationnelle-demandée-par-le-cdc) porte
l'UI et l'accès client. Aucun nouveau chantier n'est créé. Incrément estimé : **~1,5–3 j-dev** après
prise en compte des composants déjà livrés.

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

### 🟠 5. "+1 aux activités" (§3.1) — questions client préparées

Le cahier des charges autorise explicitement "s'inscrire avec un +1 aux deux activités". Aucune trace
d'un flux accompagnant n'a été trouvée dans le code. Les décisions nécessaires sont préparées pour le
[call du 21/07](./08-questions-call-lfd-2026-07-21.md) : identité/email du `+1`, groupe indivisible,
capacité de deux places, billet/QR propre, scan autonome, modification et annulation. Après réponse,
répartir l'implémentation entre H/BIL et B/C2.1/D.

### 🟠 6. Plafonds quotidiens (§3.1, §3.2) — répartis H + N-ANTI

La limite métier est maintenant rattachée à **H-D8** : maximum deux choix `confirmed` par personne et
date `Europe/Paris`, sous verrou registration, avec E2E H-T15→H-T19. `waitlisted`, `cancelled` et
`blocked` ne comptent pas selon la décision actuelle, à confirmer au call.

L'interprétation appareil/IP est rattachée à **N-ANTI**, pas à N1 : recommandation d'une limite dure
par email/personne et de l'IP comme signal anti-abus à seuil élevé, pour ne pas bloquer un NAT partagé.
La décision client est demandée au call du 21/07.

### 🟠 7. Chantier K — PCA documenté (§6.1) — 10 %

Le cahier des charges exige un "plan de continuité en cas d'incident (PCA) documenté". Le chantier K
(résilience event / checklist / runbooks) n'est qu'à 10 % dans le suivi — checklist créée, vérifications
à dérouler.

### 🔴 7 bis. Scan en conditions réelles (§4.1) — GO terrain non acquis

L'audit statique du 19/07 confirme un socle mobile utile : queue offline persistée, replay FIFO,
préchargement des sessions/historiques, feedback animé et haptique, et verrou serveur empêchant deux
lignes admises pour le même participant/session. Il révèle cependant plusieurs écarts :

- aucun son de succès/refus/doublon n'est implémenté ;
- un re-scan session renvoie l'ancien scan en HTTP 201 sans signaler le doublon au mobile ;
- la queue et le store session ne dédupliquent pas deux scans offline identiques ;
- le service mobile perd le statut et les données structurées des erreurs de scan session ;
- le contrôle de capacité est fait avant le verrou par participant : plusieurs scanners sur des QR
  différents peuvent dépasser la dernière place ;
- les registrations volumineuses ne sont pas persistées : après fermeture forcée, le cold start
  réellement offline ne garantit pas la résolution locale des QR ;
- aucun test automatisé mobile n'a été trouvé et le typecheck de la branche `main` échoue ;
- aucune répétition terrain complète ni PV GO/NO-GO n'existe.

Répartition actée : [M](./M-appli-mobile-lfd2026/README.md) possède l'app/UX/offline/appareils,
[H/L9.1](./H-inscriptions-session/tests-sessions.md#scan-concurrent-et-replay-offline--ajout-audit-du-1907)
possède le contrat doublon et la capacité atomique, et [K](./K-resilience-event/README.md#raccord-scan-terrain-m)
organise la [recette réelle](./M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md), la contre-recette
et le GO opérationnel.

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

1. **RBAC + audit O des actions BIL** (points 0 et 0 bis) — livrer ensemble les autorisations et leur
   preuve ; ne pas considérer une trace HTTP générique comme un audit métier complet.
2. **CAPTCHA/anti-bot** (point 1) — coût probablement faible (intégration reCAPTCHA/hCaptcha/Turnstile),
   impact élevé face au scénario de pics décrit dans le cahier des charges.
3. **L9.1 puis J-ENTREES** (point 2 bis) — fixer la vérité des présences, le dashboard et l'export,
   puis les inclure dans le test combiné ; ne pas valider un écran fondé sur `registered_count`.
4. **Load test réel de J** (point 2) — condition de validation explicite du cahier des charges (3000 req
   simultanées), à ne pas découvrir en prod le jour J.
5. **Décision explicite sur le SLA/HA** (point 3) — pas forcément du dev, mais une clarification/
   communication à faire rapidement plutôt qu'un silence sur la contradiction.
6. **C2.1 billet PDF** (point 4) — déjà priorisé en J2 dans le suivi, à ne pas laisser glisser.
7. **Trancher le `+1` et l'interprétation IP au call du 21/07** ; la limite deux `confirmed`/jour est
   déjà ajoutée à H, sous réserve de confirmation client.
8. **PCA (K)** et **volet RGPD contractuel** (points 7, 8) — à sortir de l'angle mort, même si ce n'est
   pas uniquement du développement.
9. **Fermer M/H scan puis répéter sur le parc réel** — plusieurs scanners, offline,
   doublons et capacité sont des gates GO/NO-GO, pas une simple smoke de fin de chantier.
