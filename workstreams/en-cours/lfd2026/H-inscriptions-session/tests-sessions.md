# Chantier H — Tests sessions/inscriptions (à livrer AVEC le chantier)

> **Statut : 🔴 LISTE À VALIDER — rien n'est codé.**
> **Owner : Corentin** (les tests suivent le code H).
> Déplacé depuis le [chantier T — tests event-critical](../T-tests-event/README.md) le 13/07 :
> ces tests dépendent des **règles métier écrites pendant H** (déjà scanné, event vs session,
> sessions privées/publiques) — impossibles à écrire avant. **La V1 du CI/CD (chantier 0-CI)
> est livrée sans**, et ces tests sont livrés **avec ou avant la livraison de H**.

- **Suivi :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **H**)
- **Chantier tests global (hors sessions) :** [../T-tests-event/README.md](../T-tests-event/README.md)

---

## 1. Règles métier à tester (rappel des décisions 13/07)

Le scan session ne suit **pas** la même logique qu'un simple refus binaire :

1. **Déjà scanné ≠ refus dur** : le re-scan ne re-valide pas l'entrée, mais la réponse doit
   indiquer clairement l'état « déjà scanné » **et l'action de re-scan doit être conservée
   dans les logs** (traçabilité : qui a re-scanné, quand, où).
   ⚠️ **Contrat actuel du back (vérifié 13/07)** : le check-in event renvoie **409
   `ALREADY_CHECKED_IN`** + état serveur (réconciliation offline mobile) et **ne logge rien**.
   À trancher pour les sessions : même contrat 409 ou 2xx — et brancher la persistance du
   re-scan (AuditLog ?) qui **manque aujourd'hui** (voir [chantier T §5bis](../T-tests-event/README.md)).
2. **Pas inscrit à l'event** → refus.
3. **Inscrit à l'event mais pas à la session** :
   - session **privée** (avec inscription) → refus ;
   - session **publique** (sans inscription — être présent à l'event suffit) → **accepté**.
4. **Droits scanner** : dépend de la logique métier définie dans H (qui a le droit de scanner
   quoi : event, session, les deux).

## 1bis. État réel du code scan (vérifié 13/07 dans `registrations.service.ts`) — point de départ pour H

| Exigence | Code actuel | Verdict |
| --- | --- | --- |
| HMAC branché sur la route de scan | `checkInByCode` → `ScanCodeResolverService` : vérif HMAC stricte, signature invalide → 400, UUID brut refusé | ✅ conforme, mais **aucun E2E** sur la route (unitaire seulement) → T2 (chantier T) |
| Déjà scanné | **409 `ConflictException`** `code: ALREADY_CHECKED_IN` + état serveur complet (pour réconciliation **offline mobile**) | ⚠️ pas un 2xx — choix délibéré mobile ; **reco : garder 409** · même décision à prendre pour le scan session |
| Re-scan conservé dans les logs | Exception levée, **rien n'est persisté**. `AuditLog` existe dans Prisma mais jamais appelé ici | 🔴 **non respecté** — dev à ajouter dans H (brancher la persistance sur le chemin « déjà scanné », event ET session) |
| Multi-scan par point de contrôle | Un seul `checked_in_at` global par inscription + checkout ; **pas de scan par session** | ⏳ c'est précisément ce que H construit (modèle de check-in par session) |

## 1ter. Code à développer (pré-requis des tests — à intégrer au découpage H)

Les tests §2 ne peuvent pas être verts sans ces devs. Chaque test référence le dev qui le rend possible.

| # | Dev à faire | Détail | Débloque les tests |
| --- | --- | --- | --- |
| H-D1 | 🔴 **Persistance du re-scan** | Brancher `AuditLog` (ou table dédiée `checkin_logs`) sur le chemin « déjà scanné » : qui a re-scanné, quand, où — pour le check-in **event** (chemin `ALREADY_CHECKED_IN` existant) **et** le scan **session** | H-T2 |
| H-D2 | **Contrat de réponse « déjà scanné » session** | Trancher 409 (aligné mobile offline, reco) vs 2xx, puis implémenter le même contrat que l'event (code erreur + état serveur) sur la route session | H-T2 |
| H-D3 | **Modèle check-in par session** | Table/relation de check-in **par point de contrôle** (session), en plus du `checked_in_at` event global — cœur du multi-scan légitime (entrée → sessions → checkout → re-checkin) | H-T1 → H-T6 |
| H-D4 | **Règle d'admission session** | Logique privée (inscription session requise) vs publique (présence event suffit) + refus si pas inscrit à l'event — idéalement extraite en **fonction pure** testable en unitaire | H-T3, H-T4, H-T5 + unitaires §2 |
| H-D5 | **Droits scanner par session/event** | Définir et implémenter qui peut scanner quoi (event, session, les deux) | H-T6, H-T14 |
| H-D6 | **CRUD sessions + mode d'inscription** | Création/modification session avec mode privé/public, ouverture/fermeture d'inscription respectée par le endpoint public | H-T7 → H-T14 |
| H-D7 | **Capacité / waitlist session** | Refus ou waitlist à capacité atteinte selon config | H-T9 |

> H-D3 à H-D7 = le cœur du chantier H déjà cadré — listés ici pour tracer le lien dev ↔ test.
> **H-D1 et H-D2 sont les ajouts issus de la vérif code du 13/07** (§1bis) : petits mais
> indispensables pour la règle « garder l'action du re-scan ».

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

### Unitaires (logique pure, si extraite en services)

- Règle d'admission (event/session/privée/publique/déjà scanné) isolée en fonction pure → table de vérité.
- Calcul capacité/waitlist.

## 3. Intégration CI

- Ces tests rejoignent la **même suite E2E** que le chantier T (`test/*.e2e-spec.ts`,
  supertest + Postgres/Redis réels) — ils tournent à chaque PR comme le reste
  (voir [stratégie CI, chantier T §4bis](../T-tests-event/README.md)).
- Deviennent **bloquants** (required check) à la livraison de H.

## 4. Definition of done

- H-D1 → H-D7 livrés (ou explicitement descopés avec trace ici) — en particulier **H-D1
  (persistance re-scan)** qui manque aujourd'hui dans le code.
- H-T1 → H-T14 verts en CI (ou explicitement descopés avec trace ici).
- La règle « re-scan loggé » vérifiée par un test qui lit réellement le log/l'historique.
- Liste validée par Rabie + Corentin avant d'écrire le premier test.
