# Chantier H — Tests sessions/inscriptions (à livrer AVEC le chantier)

> **Statut au 24/07 : 🟢 H-T1→H-T19 livrés ; 24/24 E2E H verts localement.**
> H-T20→H-T25 sont couverts par les suites de cohérence L9.1 et la contre-recette M/K ; la preuve
> d'audit détaillée appartient au chantier O.
> **Owner : Corentin** (les tests suivent le code H).
> Déplacé depuis le [chantier T — tests event-critical](../T-tests-event/README.md) le 13/07 :
> ces tests dépendent des **règles métier écrites pendant H** (déjà scanné, event vs session,
> sessions privées/publiques). La suite actuelle couvre le nominal et les refus principaux, mais
> l'audit mobile/terrain du 19/07 ajoute la concurrence multi-scanner, la réconciliation offline et
> la cohérence des métriques utilisées par J-ENTREES/BIL.

- **Suivi :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **H**)
- **Chantier tests global (hors sessions) :** [../T-tests-event/README.md](../T-tests-event/README.md)

---

## 1. Règles métier à tester (rappel des décisions 13/07)

Le scan session ne suit **pas** la même logique qu'un simple refus binaire :

1. **Déjà scanné ≠ refus dur** : le re-scan ne re-valide pas l'entrée, mais la réponse doit
   indiquer clairement l'état « déjà scanné » **et l'action de re-scan doit être conservée
   dans les logs** (traçabilité : qui a re-scanné, quand, où).
   ⚠️ **Contrat réel vérifié le 19/07 :** le check-in event renvoie 409
   `ALREADY_CHECKED_IN`, mais le scan session renvoie l'ancien scan en **HTTP 201 sans indicateur
   de doublon**. Le mobile affiche alors un nouveau succès. Le contrat session doit devenir explicite
   et l'action de re-scan doit être auditée via O.
2. **Pas inscrit à l'event** → refus.
3. **Inscrit à l'event mais pas à la session** :
   - session **privée** (avec inscription) → refus ;
   - session **publique** (sans inscription — être présent à l'event suffit) → **accepté**.
4. **Droits scanner** : dépend de la logique métier définie dans H (qui a le droit de scanner
   quoi : event, session, les deux).
5. **Plusieurs scanners** : deux appareils sur le même participant produisent une seule admission ;
   plusieurs participants concurrents ne doivent jamais dépasser la capacité de check-in.

## 1bis. État réel du code scan (vérifié 19/07)

| Exigence | Code actuel | Verdict |
| --- | --- | --- |
| HMAC route session | `ScanCodeResolverService` vérifie `code` côté serveur ; E2E H-T1→H-T5 présents | ✅ socle présent |
| Re-scan même session | advisory lock par `sessionId:registrationId`, puis retour de l'ancien `SessionScan` en HTTP 201 | 🟠 une seule ligne serveur, mais aucun signal doublon pour le mobile |
| Re-scan conservé dans l'audit | aucun événement explicite sur le chemin idempotent | 🔴 à porter par [O](../O-audit-lfd/README.md), pas par un stockage H parallèle |
| Capacité multi-scanner | `COUNT(IN)`/`COUNT(OUT)` avant la transaction ; verrou ensuite par participant | 🔴 course possible entre deux participants sur la dernière place |
| Historique par session | table `session_scans` avec IN/OUT/status, endpoint history et store mobile | ✅ présent |
| E2E concurrents scan | aucun `Promise.all` couvrant même QR ou dernière place | 🔴 absent |

## 1ter. Code à développer (pré-requis des tests — à intégrer au découpage H)

Les tests §2 ne peuvent pas être verts sans ces devs. Chaque test référence le dev qui le rend possible.

| # | Dev à faire | Détail | Débloque les tests |
| --- | --- | --- | --- |
| H-D1 | 🔴 **Émission du re-scan** | Émettre l'événement explicite défini par O avec auteur/appareil, session, participant, premier scan et tentative ; O possède le stockage/audit | H-T2 + O-TEST |
| H-D2 | **Contrat « déjà scanné » session** | Recommandation : 409 `ALREADY_SESSION_SCANNED` + scan/registration/heure serveur, aligné avec le check-in event ; le mobile doit préserver `status/data` | H-T2, H-T20, H-T22 |
| H-D3 | ✅ **Modèle check-in par session** | `session_scans` IN/OUT/status et historique par session présents | H-T1 → H-T6 |
| H-D4 | **Règle d'admission session** | Logique privée (inscription session requise) vs publique (présence event suffit) + refus si pas inscrit à l'event — idéalement extraite en **fonction pure** testable en unitaire | H-T3, H-T4, H-T5 + unitaires §2 |
| H-D5 | **Droits scanner par session/event** | Définir et implémenter qui peut scanner quoi (event, session, les deux) | H-T6, H-T14 |
| H-D6 | **CRUD sessions + mode d'inscription** | Création/modification session avec mode privé/public, ouverture/fermeture d'inscription respectée par le endpoint public | H-T7 → H-T14 |
| H-D7 | **Capacité / waitlist session** | Refus ou waitlist à capacité atteinte selon config | H-T9 |
| H-D8 | ✅ **Maximum deux activités confirmées/jour** | Verrou registration, jour `Europe/Paris`, seuls les `confirmed` comptés ; troisième refusé sans toucher la jauge | H-T15 → H-T19 ✅ |
| H-D9 | 🔴 **Capacité atomique au scan** | Déplacer la décision de capacité dans une transaction/verrou commun à la session ou utiliser le compteur atomique L9.1 ; ne pas verrouiller seulement le participant | H-T21 |
| H-D10 | **Contrat de replay idempotent** | La même mutation rejouée doit retrouver son résultat sans créer une deuxième admission et doit permettre au mobile d'afficher le doublon | H-T20, H-T22 |

> H-D9/H-D10 sont des dépendances directes de la recette M. Leur effort reste dans H/L9.1 et ne doit
> pas être compté une seconde fois dans M.

## 2. Liste des tests proposés — ✋ À VALIDER

### Scan session (E2E API)

| # | Cas | Attendu |
| --- | --- | --- |
| H-T1 | QR valide, inscrit event + session privée | accepté, check-in enregistré |
| H-T2 | Re-scan du même QR sur la même session | état « déjà scanné » explicite (contrat 409 ou 2xx à trancher) + **action loggée** |
| H-T3 | QR valide mais user **pas inscrit à l'event** | refus |
| H-T4 | Inscrit event, **pas inscrit** à une session **privée** | refus |
| H-T5 | Inscrit event, session **publique** (sans inscription) | accepté |
| H-T6 | User sans droit scanner sur cette session/event | refus (selon logique métier H) |

### Inscriptions (E2E API — ex-T4 du chantier T)

| # | Cas | Attendu |
| --- | --- | --- |
| H-T7 | Création inscription nominale (lien public) | inscrit + QR/badge généré selon flow |
| H-T8 | Doublon (même email / même personne) | géré proprement (pas de double inscription) |
| H-T9 | Capacité atteinte | refus ou waitlist selon config session |
| H-T10 | Fermeture d'inscription → tentative d'inscription | refus |

### CRUD sessions (E2E API + unitaires sur la logique)

| # | Cas | Attendu |
| --- | --- | --- |
| H-T11 | Création session (privée / publique) | créée avec le bon mode d'inscription |
| H-T12 | Modification session (horaires, capacité, mode) | persistée · cohérence avec inscriptions existantes |
| H-T13 | Ouverture/fermeture d'inscription | état respecté par le endpoint public |
| H-T14 | Permissions CRUD (qui peut créer/modifier/fermer) | selon rôles définis dans H |

### Limite CDC — deux activités confirmées par jour

| # | Cas | Attendu |
| --- | --- | --- |
| H-T15 | Première puis deuxième session `confirmed` le même jour Europe/Paris | deux choix acceptés |
| H-T16 | Troisième session qui serait `confirmed` le même jour | refus métier, aucun choix créé, compteur session inchangé |
| H-T17 | Deux sessions confirmées vendredi puis une session samedi | session du lendemain acceptée |
| H-T18 | Choix `waitlisted`, `cancelled` ou `blocked` le même jour | ne consomme pas le quota confirmé ; annulation libère le quota |
| H-T19 | Trois inscriptions concurrentes vers trois sessions du même jour | au maximum deux choix `confirmed`, aucune course ni sur-comptage |

### Scan concurrent et replay offline — ajout audit du 19/07

| # | Cas | Attendu |
| --- | --- | --- |
| H-T20 | N requêtes concurrentes, même QR et même session | une seule ligne `admitted`, les autres réponses indiquent explicitement le doublon |
| H-T21 | N QR différents sur une session avec une seule place restante | exactement une admission supplémentaire, capacité jamais dépassée |
| H-T22 | même QR mis en queue offline par deux appareils puis rejoué | une admission, une réconciliation doublon exploitable par M, aucun faux succès |
| H-T23 | erreur métier pendant replay session | statut/code/état serveur conservés jusqu'au mobile, mutation classée sans trois faux retries réseau |

### Compteurs de présence — prérequis de J-ENTREES/BIL

| # | Cas | Attendu |
| --- | --- | --- |
| H-T24 | `IN` admis puis `OUT` admis | présence courante revient à 0 et visiteur unique reste à 1 |
| H-T25 | scan rejeté ou doublon/replay | ni `present_count` ni le nombre de visiteurs uniques ne sont sur-comptés |

Le calcul du taux, la propagation live, l'export nominatif et ses droits sont prouvés dans les tests
`J-E1`→`J-E6` de [J-ENTREES](../J-capacite-live/README.md#tests-de-sortie), pas dans H.

### Unitaires (logique pure, si extraite en services)

- Règle d'admission (event/session/privée/publique/déjà scanné) isolée en fonction pure → table de vérité.
- Calcul capacité/waitlist.

## 3. Intégration CI

- Ces tests rejoignent la **même suite E2E** que le chantier T (`test/*.e2e-spec.ts`,
  supertest + Postgres/Redis réels) — ils tournent à chaque PR comme le reste
  (voir [stratégie CI, chantier T §4bis](../T-tests-event/README.md)).
- Deviennent **bloquants** (required check) à la livraison de H.

## 4. Definition of done

- H-D1 → H-D10 livrés (ou explicitement descopés avec trace ici).
- H-T1 → H-T25 verts en CI (ou explicitement descopés avec trace ici).
- La règle « re-scan loggé » vérifiée par un test qui lit réellement le log/l'historique.
- La capacité et le doublon sont ensuite contre-recettés sur appareils dans M/K.
