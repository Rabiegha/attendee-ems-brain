# Chantier H — Inscriptions par session

> Lien public par session + capacité/waitlist + refonte front. **Plus gros morceau restant.**
> Cadré dans un [workstream dédié](../../../a-faire/sessions-inscriptions-lfd2026/README.md).

## Mise à jour du 24/07 — H terminé et recetté staging

- ✅ Socle session public/admin, capacité, waitlist, compteurs et ouverture programmée.
- ✅ Contrat de doublon `409 ALREADY_SESSION_SCANNED`, concurrence même QR et dernière place.
- ✅ Maximum deux sessions `confirmed` par jour `Europe/Paris`, appliqué dans la frontière
  transactionnelle commune à toutes les routes.
- ✅ H-T1→H-T19 : **24/24 E2E verts** sur PostgreSQL/Redis locaux, dont trois inscriptions
  concurrentes et waitlist hors quota.
- ✅ Front V1 mode-aware : le mode walk-in masque toute donnée d'inscription et affiche uniquement
  les présences ; le mode avec inscription expose inscrits, capacité, waitlist et contrôles.
- ✅ Builds back/front et typecheck front verts.
- ✅ PR back #68 et front #27 mergées, CI/CD staging verts ; backend servi au SHA
  `245427cdf678ba4879ecfa6b0c523c05c2ec18f9`.
- ✅ Recette API staging create-only : **52/52 contrôles verts** sur l'événement isolé
  `H-RECETTE-20260724122829` (`dda8c24f-6b79-4156-b985-f73db2d47aab`).
- ✅ Aucun `DELETE`, seeder, reset, truncate ni écriture SQL ; aucun événement existant modifié.
- ℹ️ La contre-recette appareils appartient à M/K, l'audit détaillé à O et la promotion production
  au run de livraison : ces opérations ne rouvrent pas le développement H.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (§3-H)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **H**)
- **Statut :** ✅ 100 % — code, merge, CI/CD et recette staging terminés.
- **Détail avancement :** [workstream dédié — §Avancement](../../../a-faire/sessions-inscriptions-lfd2026/README.md)
- **Tests à livrer avec le chantier :** [tests-sessions.md](./tests-sessions.md) — socle H-T1→H-T14,
  quota H-T15→H-T19, concurrence/replay H-T20→H-T23 et cohérence compteur H-T24→H-T25.

## Source de vérité des entrées pour J-ENTREES

H/L9.1 fournit la vérité métier au
[tableau de bord des entrées](../J-capacite-live/README.md#j-entrees--tableau-de-bord-des-entrées-lfd) :

- `registered_count` reste le nombre de réservations `confirmed` et ne prouve aucune entrée ;
- `present_count` est la présence physique actuelle, basée sur le dernier scan admis IN/OUT ;
- la participation post-event est vraie dès qu'il existe au moins un `IN admitted`, même si la
  personne a ensuite scanné `OUT` ;
- le remplissage utilise `checkin_capacity ?? capacity`, jamais la seule capacité d'inscription ;
- doublons et scans rejetés ne modifient aucun compteur de présence.

La diffusion live, le dashboard et l'export appartiennent respectivement à J-ENTREES et BIL. Le
compteur atomique reste compté une seule fois dans L9.1.

## Ajout CDC du 19/07 — maximum deux activités confirmées par jour

### Règle retenue

Pour une même inscription/personne, Attendee autorise au maximum **deux choix de session au statut
`confirmed` par date civile dans le fuseau `Europe/Paris`**.

- la date est calculée depuis `sessions.start_at` en `Europe/Paris`, jamais avec la date UTC brute ;
- seuls les choix `confirmed` consomment la limite ;
- `waitlisted`, `cancelled` et `blocked` ne consomment pas la limite ;
- une réinscription idempotente à une session déjà choisie ne consomme pas une deuxième place ;
- une annulation libère la possibilité de confirmer une autre activité le même jour ;
- le lendemain, un nouveau quota de deux activités est disponible ;
- la règle s'applique à tous les endpoints capables d'ajouter des choix de session, pas seulement au
  formulaire BIL, afin d'éviter un contournement par une autre route.

### Contrainte de concurrence

Le contrôle doit vivre dans la transaction et sérialiser les ajouts pour une même registration.
Un simple `COUNT` avant transaction est insuffisant : trois demandes simultanées vers trois sessions
différentes pourraient toutes voir une place disponible dans le quota personnel.

Ordre métier attendu :

```text
identifier/créer la registration event
  -> verrouiller la registration pour ce participant
  -> si choix cible déjà présent : retour idempotent
  -> compter les choix confirmed du même jour Europe/Paris
  -> si compteur >= 2 : refuser sans toucher la jauge session
  -> appliquer ensuite capacité/waitlist et créer le choix
```

Si la session cible part en `waitlisted`, elle ne consomme pas le quota confirmé selon la décision
actuelle. Cette interprétation doit être confirmée au call client du 21/07.

### Estimation ajoutée à H

- implémentation transactionnelle sur tous les chemins : **0,5–1 j** ;
- E2E, concurrence et dates/fuseau : **0,5–1 j** ;
- ajout H total : **1–2 j-dev**.

L'estimation H passe provisoirement de **~5,5–8,5 j à ~6,5–10,5 j-dev**.

## Dépendance M du 19/07 — doublon session et multi-scanner

### Problème actuel

Le serveur protège déjà partiellement les données : un advisory lock sur
`sessionId:registrationId` sérialise deux scans du même participant et le second appel retrouve la
ligne `SessionScan` existante. Il ne crée donc normalement pas une seconde admission.

Le problème est le **contrat de réponse** : le second appel renvoie cette ancienne ligne en HTTP 201,
exactement comme une nouvelle admission. Le téléphone ne peut pas savoir s'il vient d'enregistrer une
entrée ou s'il a retrouvé une entrée réalisée auparavant par un autre scanner. Il affiche donc un
succès vert trompeur.

Le cas offline rend ce défaut plus visible :

```text
téléphone A offline : QR-123 -> queue A
téléphone B offline : QR-123 -> queue B
retour réseau :
  A rejoue -> admission créée
  B rejoue -> ancienne admission renvoyée comme nouveau succès 201
```

Le serveur reste l'autorité entre appareils. M peut supprimer deux mutations identiques présentes sur
un même téléphone, mais il ne peut pas connaître la queue d'un autre téléphone avant la reconnexion.

### Contrat serveur attendu pour M

Recommandation : aligner le scan session sur le check-in event et retourner un conflit structuré :

```json
{
  "code": "ALREADY_SESSION_SCANNED",
  "message": "Participant déjà scanné sur cette session",
  "duplicate": true,
  "session_scan": {
    "id": "uuid",
    "scan_type": "IN",
    "scanned_at": "date serveur",
    "status": "admitted"
  },
  "registration": {}
}
```

Statut recommandé : **409**. Ce n'est pas une panne à retenter : M retire la mutation de sa queue,
fait un upsert de `session_scan`, réconcilie l'état local et affiche « déjà scanné à HH:mm ».

### Travail H/L9.1 à livrer

| Lot | Travail serveur | Preuve |
| --- | --- | --- |
| H-M1 | retourner `ALREADY_SESSION_SCANNED` avec scan, registration et heure serveur | H-T2, H-T20 |
| H-M2 | conserver une seule ligne admise sous N requêtes du même QR | H-T20 |
| H-M3 | émettre la tentative de re-scan selon le contrat d'audit O | H-T2 + O-TEST |
| H-M4 / L9.1 | déplacer la capacité dans une opération/transaction atomique au niveau session | H-T21 |
| H-M5 | garantir un résultat de replay exploitable, sans transformer le doublon en erreur réseau | H-T22, H-T23 |

La capacité est un problème distinct du doublon : le verrou actuel est par participant. Deux QR
**différents** sur la dernière place ne partagent donc pas ce verrou et peuvent être admis ensemble si
les `COUNT` ont été lus simultanément. L9.1 doit corriger cela avec `present_count` et une garde
atomique au niveau session.

### Frontière de responsabilité

- **H/L9.1 :** vérité serveur, unicité, contrat 409, capacité atomique et E2E concurrents ;
- **M :** déduplication de la queue d'un appareil, conservation `status/data`, upsert local et feedback ;
- **K :** vérification sur plusieurs appareils à J-10 puis J-4.

Ces garanties font partie de la Definition of Done de H pour LFD. Elles sont détaillées dans
[tests-sessions.md](./tests-sessions.md#scan-concurrent-et-replay-offline--ajout-audit-du-1907) et
contre-recettées via [M](../M-appli-mobile-lfd2026/recette-terrain-scan-lfd.md). Le contrat doublon
était déjà dans le périmètre scan H ; la capacité atomique est comptée dans L9.1, donc aucun jour ne
doit être ajouté deux fois.
