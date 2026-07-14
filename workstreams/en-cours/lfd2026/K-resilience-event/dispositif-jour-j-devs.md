# Dispositif jour J — présence des devs (proposition)

> Chantier K — Résilience event LFD 2026 (4-5 septembre, Sorbonne Nouvelle, ~10 000 participants/jour).
> Contexte : la manager souhaite les 2 devs **sur place**. Cette note propose un dispositif
> qui couvre les vrais risques du jour J, à discuter avec elle.

## Status

- **Type :** DRAFT — proposition à valider avec la manager
- **Créé :** 2026-07-14

---

## 1. Analyse des risques réels du jour J (spécifique LFD)

**Particularité LFD : PAS d'impression de badges sur place** (billets numériques, QR dans le mail).
Le poste « physique » classique (imprimantes, print-client) n'existe pas pour cet event.

| Risque | Nature | Où ça se gère |
| --- | --- | --- |
| 🔴 **Pics d'inscription jour même** — créneaux ouvrant à 9h/11h/14h, ≥3 000 connexions simultanées (exigence cahier des charges MEAE) | Backend pur (capacité, portier Redis, DB) | **War-room / à distance** : SSH, dashboards, hotfix CI/CD |
| 🔴 API down / 5xx / saturation | Backend | War-room |
| 🟡 App de scan (bénévoles + agents MEAE, mode hors-ligne exigé) | Terrain : formation, assistance, remplacement d'appareil | **Sur place** |
| 🟡 Wifi/4G du lieu saturé pour les scans | Terrain + réseau | Sur place (constat) + backend (mode offline) |
| 🟡 Fermeture de jauge / ajustement de créneaux en urgence | Back-office | Les deux (back-office accessible partout) |
| ⚪ Astreinte SLA 99,9 % exigée (J-15 → J+1) | Contractuel | Nécessite quelqu'un **en capacité d'intervenir serveur** |

**Conclusion de l'analyse** : sans impression, le risque dominant est **backend** (pics
d'inscription du jour même). Or un incident backend ne se diagnostique pas debout dans un hall
avec le wifi saturé par 10 000 personnes — l'incident 502 prod du 13/07 (7 min, SSH + logs + sang-froid)
en est la démonstration récente.

## 2. Dispositif recommandé — « 1 + 1 »

- **1 dev sur place** : référent scan/terrain (assistance bénévoles, constats réseau, lien organisation),
  avec laptop prêt à opérer en secours.
- **1 dev en war-room à distance** (chez lui ou bureau) : dashboards, SSH, CI/CD, hotfix,
  astreinte SLA. Environnement calme + connexion fiable.
- Rotation possible entre les deux jours (vendredi/samedi) pour l'équité.

**Argument pour la manager** : « on couvre les deux fronts au lieu d'un seul — la présence sur
place n'a de valeur que si quelqu'un peut réellement réparer le serveur pendant ce temps ».

## 3. Si les 2 devs doivent être sur place — conditions non négociables

1. **Routeur 4G/5G dédié** (jamais le wifi du lieu), testé en amont, + un 2e opérateur en secours.
2. **Coin war-room au calme** sur le site : table, prises, si possible écran.
3. Les 2 laptops **prêts à opérer**, testés la veille : SSH VPS **par clé**, `gh` CLI authentifié,
   accès Netdata + Sentry, checkout des repos à jour.
4. **Runbooks chantier K finalisés** (incidents types : API down, pic capacité, DB saturée,
   rollback — avec commandes exactes).
5. **Chemin hotfix rodé** : CD prod `hotfix=true` (bypass fenêtre 22h-06h) + rollback auto —
   ✅ déjà validés en réel le 13/07.
6. **`NOTIFY_WEBHOOK` branché** : alertes deploy + monitoring sur les téléphones.
7. Numéros utiles imprimés : OVH support, contacts MEAE/organisation, hébergeur DNS.

## 4. Reste à faire pour rendre ce dispositif crédible (dépendances)

- [ ] Runbooks K (les vrais, testés) — le chantier est à 10 %
- [ ] Déploiement 0-MON complet (Netdata + alertes + webhook) — code prêt, pas déployé
- [ ] Test de charge validant les pics d'ouverture de créneaux (chantiers I/J)
- [ ] Décision manager sur le dispositif (1+1 vs 2 sur place avec conditions)
