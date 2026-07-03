# LFD 2026 — Résumé exécutif (version client)

> Document de synthèse, destiné à être partagé avec le client institutionnel.
> Évènement diplomatique public — 4 et 5 septembre 2026.
> Rédigé le 22 mai 2026.

---

## 1. Ce que couvre la plateforme Attendee EMS

La plateforme Attendee EMS est une solution de gestion d'évènements multi‑tenant déjà éprouvée
en production. Elle couvre l'intégralité du parcours visiteur demandé au cahier des charges :

| Fonctionnalité demandée | Couverture | Détail |
|---|---|---|
| Landing page publique d'inscription | ✅ Présent | Frontend React/Vite, mobile‑friendly |
| Sélection activité / session / jauge | ✅ Présent | Gestion capacités par espace et créneau |
| Formulaire court (nom, prénom, email) | ✅ Présent | Champs configurables |
| Création inscription | ✅ Présent | API NestJS + PostgreSQL |
| Génération badge PDF + QR code | ✅ Présent | Puppeteer + librairie qrcode |
| Email de confirmation avec badge | ✅ Présent | SMTP via nodemailer |
| Check‑in mobile (scan QR) | ✅ Présent | App React Native iOS + Android |
| Validation badge (valide / déjà utilisé) | ✅ Présent | Unicité garantie DB |
| Impression badges sur place | ✅ Présent | Print client Electron + Socket.IO |
| Dashboard temps réel | ✅ Présent | WebSocket namespace `/events` |
| HTTPS | ✅ Présent | Nginx + Let's Encrypt |
| Compatibilité iOS / Android | ✅ Présent | App native + landing responsive |
| Journalisation accès et actions back‑office | ✅ Présent | Logs Pino + Sentry |

## 2. Ce que nous nous engageons à tester avant l'évènement

| Test | Date cible | Critère |
|---|---|---|
| Capacité **3 000 requêtes simultanées** à l'ouverture | J‑10 (25 août 2026) | ≥ 95 % succès, p95 < 2,5 s |
| Test d'endurance 1 h à charge soutenue | J‑7 | Latence stable, 0 régression |
| Spike créneaux 09h / 11h / 14h | J‑8 | File d'attente drainée < 2 min |
| Anti‑surbooking (test concurrentiel) | J‑15 | 0 surbooking, garanti par verrou DB |
| Check‑in massif (500 scans/min) | J‑12 | p95 < 600 ms, pas de double scan |
| Restauration complète de la base | J‑20 | RTO mesuré < 30 min |

Les rapports de tests seront transmis au client au plus tard à J‑7, avec verdict go/no‑go.

## 3. Infrastructure recommandée

Deux options sont proposées (détaillées dans `lfd-2026-capacity-planning.md`) :

| Option | Objectif | SLA atteignable | Complexité | Recommandation |
|---|---|---|---|---|
| **A — Minimale renforcée** | Sécuriser sans sur‑ingénierie | 99,5 % global, 99,9 % sur J et J+1 | Faible à moyenne | Si contrainte budgétaire |
| **B — Robuste scaling horizontal** | Tenir 99,9 % strict sur J‑15 → J+1 | 99,9 % garanti | Moyenne à élevée | **Recommandée** |

Les deux options incluent :
- API NestJS avec génération badge et envoi email **passés en file asynchrone** (BullMQ + Redis).
- PostgreSQL managé avec backups quotidiens et restore testé.
- Reverse proxy Nginx + WAF/anti‑bot Cloudflare devant.
- Stockage badges PDF chiffré sur Cloudflare R2.
- Monitoring Sentry + uptime externe + alertes 24/7.
- Astreinte technique dédiée 24/7 sur les 4 et 5 septembre.

L'option B ajoute :
- Auto‑scaling horizontal des API et workers.
- Failover régional PostgreSQL.
- Point‑in‑time recovery (RPO 5 minutes).
- Salle d'attente virtuelle Cloudflare en cas de pic extrême.
- Environnement de staging permanent iso‑production.

## 4. Engagements proposés au client

| Engagement | Valeur |
|---|---|
| Disponibilité (J‑15 → J+1) | **99,9 %** (option B) — soit ≤ 24 min d'indispo sur 17 jours |
| Capacité de pic | **3 000 requêtes simultanées** validée par test |
| Latence p95 sur l'inscription | < 1,5 s en charge normale, < 2,5 s en pic |
| HTTPS, journaux, compatibilité mobile iOS/Android | Garantis (déjà en place) |
| Protection anti‑bot / anti‑scraping | Captcha + WAF Cloudflare + rate limit |
| Délai prise en charge incident critique (P1) | < 15 min, 24/7 sur J et J+1 |
| Délai résolution ou contournement P1 | < 1 h |
| RTO (reprise après sinistre majeur) | < 30 min |
| RPO (perte de données max) | 5 min (option B) / 1 h (option A) |
| Astreinte technique | 2 ingénieurs joignables 24/7 sur J et J+1 |
| Gel des déploiements | J‑15 → J+1, sauf hotfix critique validé |
| Compte rendu d'incident post‑mortem | Sous 48 h |
| Rapport SLA mensuel | Hebdomadaire pendant la fenêtre critique |

## 5. Plan de Continuité d'Activité (PCA)

Un PCA complet est documenté dans `lfd-2026-pca.md`. Il couvre :

- **13 scénarios d'incident** (backend KO, DB lente, SMTP KO, surbooking, bots, app scan offline,
  print KO, WebSocket KO, déploiement raté, incident sécurité, etc.).
- Procédures de réponse pour chacun, avec **mode dégradé** acceptable.
- Procédure de **backup et restore** chiffrée, testée à J‑20.
- **Checklists** avant ouverture, jour J, post‑évènement.
- Modèle de **communication client** par sévérité (P1 / P2 / P3).
- Roulement et **contacts d'astreinte**.

## 6. Modes dégradés acceptables (extrait)

| Composant en panne | Service maintenu | Comment |
|---|---|---|
| SMTP | Inscription OK | Badge téléchargeable immédiatement depuis la page de confirmation, email envoyé plus tard |
| Génération PDF | Inscription OK | Badge texte simple + QR généré côté frontend |
| API en surcharge | Site OK | Cloudflare Waiting Room "vous êtes le n°X" |
| WebSocket | Dashboard OK | Rafraîchissement par polling 5 s |
| App mobile offline | Check‑in OK | Cache local + synchronisation à la reconnexion |
| Print client KO | Check‑in OK | Impression manuelle depuis le PDF |

## 7. Points à confirmer avec le client

1. **Hébergement** : GCP / Scaleway / OVH / on‑prem ministère ? — impacte le choix d'option.
2. **Volumétrie email** : provider SMTP cible, autorisation d'envoi de masse, présence d'un
   secondaire (SES, Mailjet) accepté ?
3. **PDF officiel du cahier des charges** : nous demandons la version finale pour figer les jauges
   exactes par espace et par créneau (les chiffres utilisés sont ceux du prompt).
4. **Champs du formulaire d'inscription** : confirmer s'il y a des champs additionnels au‑delà de
   nom / prénom / email (téléphone, fonction, accompagnant, accessibilité…).
5. **Captcha** : préférence pour hCaptcha, reCAPTCHA, ou Cloudflare Turnstile (notre recommandation,
   intégré dans l'option WAF) ?
6. **RGPD** : DPO référent côté client, durée de conservation des données souhaitée, registre des
   traitements partagé ?
7. **WAF / CDN imposé** : présence d'un dispositif type ANSSI à respecter ?
8. **Acceptation du gel des déploiements** : du 20 août au 6 septembre, aucun changement
   applicatif sauf hotfix conjointement validé.
9. **Communication d'incident** : canal préféré (WhatsApp dédié, email, téléphone), nombre de
   contacts à alerter, langue ?
10. **Périmètre géographique** : utilisateurs FR uniquement ou international ? Latence acceptable
    hors UE ?
11. **Validation du PCA** : les modes dégradés proposés sont‑ils acceptables (notamment "badge
    téléchargeable immédiatement à la place de l'email" et "salle d'attente virtuelle en cas de
    pic > 3 000") ?
12. **Astreinte client** : un contact référent client est‑il joignable 24/7 sur J et J+1 pour
    validation des actions critiques (rollback, fermeture créneau, communication publique) ?

---

## Documents associés

- [Planification de capacité et cartographie des flux](lfd-2026-capacity-planning.md)
- [Stratégie SLA 99,9 %](lfd-2026-sla-strategy.md)
- [Plan de Continuité d'Activité (PCA)](lfd-2026-pca.md)
- [Plan de tests de charge](lfd-2026-load-test-plan.md)
