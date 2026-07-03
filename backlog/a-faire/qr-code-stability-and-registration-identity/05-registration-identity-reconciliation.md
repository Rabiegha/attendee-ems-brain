# 05 — Registration Identity Reconciliation

> **STATUS : TO REVIEW / DISCOVERY**

## 1. Le concept central

Le QR code ne doit pas dépendre uniquement de `registration.id`, qui est un identifiant **technique** susceptible de disparaître.

Il faut introduire une **identité stable d'inscription** qui survit aux opérations administratives :

- modification de données ;
- réimport partiel ou complet ;
- suppression + recréation ;
- changement de `registration.id` ou `attendee.id`.

Cette clé est **proposée** sous le nom de travail :

- `stableIdentityKey`
- ou `registrationStableKey`
- ou `externalRegistrationKey`

Le nom définitif sera tranché au design d'implémentation. Le reste du document utilise `stableIdentityKey`.

## 2. Sources possibles pour `stableIdentityKey`

Par ordre de fiabilité décroissante :

| Rang | Source | Description | Stabilité | Disponibilité |
|---|---|---|---|---|
| 1 | `external_reference` | Identifiant explicite fourni par le client (CRM, SI) | 🟢 Très haute | 🟡 Variable selon client |
| 2 | `employee_id` | Identifiant RH interne du client | 🟢 Haute | 🟡 Variable |
| 3 | `registration_number` | Numéro d'inscription métier explicite | 🟢 Haute | 🟡 Variable |
| 4 | `source_attendee_id` | ID de l'attendee dans le système source (avant import) | 🟢 Haute | 🟡 Variable |
| 5 | `normalized_email` | Email lowercased + trim + déduplication aliases | 🟡 Moyenne | 🟢 Quasi-toujours présent |
| 6 | `normalized_phone` | Téléphone normalisé E.164 | 🟡 Moyenne | 🟠 Souvent absent |
| 7 | `name + company` (fuzzy) | Combinaison nom + entreprise | 🔴 Faible | 🟢 Toujours présent |

> **Règle de design** : ne **jamais** utiliser `name + company` comme clé primaire de matching. Uniquement en **fallback** déclenchant un `needs_review`.

## 3. Stratégie de matching à l'import

Algorithme proposé pour chaque ligne d'import :

```
1. Si la ligne fournit external_reference :
     match = registration WHERE event_id = X AND external_reference = Y
     → si exactement 1 match : UPDATE
     → si 0 match : CREATE (en stockant external_reference)
     → si >1 match : CONFLICT → needs_review

2. Sinon si la ligne fournit employee_id :
     match = registration WHERE event_id = X AND employee_id = Y
     → idem

3. Sinon si la ligne fournit email :
     match = registration WHERE event_id = X AND normalized_email = lower(trim(Y))
     → si exactement 1 match : UPDATE
     → si 0 match : CREATE
     → si >1 match : CONFLICT → needs_review

4. Sinon si la ligne fournit téléphone normalisé :
     match = registration WHERE event_id = X AND normalized_phone = Y
     → idem

5. Sinon (fallback risqué) :
     fuzzy match sur (last_name + first_name + company)
     → toujours marqué needs_review, jamais auto-update.

6. Sinon :
     CREATE new registration.
```

## 4. Règles de réconciliation

### Si match fiable trouvé

- **UPDATE** la registration existante.
- **Conserver** le `QrCredential` associé : aucun nouveau QR n'est généré.
- Logger une trace `import.update` pour audit.

### Si match ambigu (multiple)

- **Ne pas écrire**.
- Créer un enregistrement de conflit (`ImportConflict` ou équivalent).
- Marquer `status = needs_review`.
- Remonter dans le rapport d'import post-traitement.

### Si registration présente en base mais absente du nouvel import

- **Ne pas hard-deleter automatiquement**.
- Proposer à l'organisateur une action explicite :
  - `keep_as_is` (rien faire) ;
  - `cancel` (status `cancelled`, QR `revoked`) ;
  - `archive` (status `archived`, QR conservé en consultation).
- **Jamais** de bouton « tout supprimer » qui hard-delete les registrations avec QR émis.

### Si QR déjà émis ou imprimé

- La registration est **protégée** contre le hard delete (cf. Option D dans [04-solution-options.md](./04-solution-options.md)).
- Seul un soft cancel est possible, qui révoque le `QrCredential` proprement.

### Si la registration a été recréée malgré tout

- Le `QrCredential` ancien peut être résolu via `event_id + stable_identity_key` vers la nouvelle registration.
- Comportement : `alias_resolved` au scan (cf. [07-scan-behavior.md](./07-scan-behavior.md)).

## 5. Cas difficiles à anticiper

### 5.1 Email changé entre deux imports

- Si `external_reference` ou `employee_id` présent : match conservé, pas de casse.
- Sinon : **aucun match fiable** → l'algorithme va créer une nouvelle registration et marquer l'ancienne comme « absente du nouvel import ».
- Solution : le rapport d'import doit signaler ces cas pour permettre à l'organisateur de corriger.

### 5.2 Doublon email dans le même import

- Détecter **avant** écriture.
- Refuser le traitement automatique → `needs_review`.
- Ne **jamais** générer deux `QrCredential` actifs pour le même email/event sans décision explicite.

### 5.3 Deux employés homonymes (même prénom, nom, entreprise)

- Sans `external_reference`, `employee_id` ou email distinct : impossible à départager.
- Marquer `needs_review` lors de l'import.
- Le scan d'un QR commun retournera `supervisor_review`.

### 5.4 Entreprise modifiée entre imports

- Sans impact si email/external_reference conservé.
- Avec changement simultané d'email + entreprise + nom : risque de doublon non détecté → `needs_review` au minimum.

### 5.5 Aucune clé stable fournie

- Cas du fichier Excel minimal : nom, prénom, email.
- `normalized_email` devient la clé de matching de fait.
- Documenter explicitement au client : « sans `external_reference`, l'email est votre garantie d'identité ».

### 5.6 Import sans colonne unique

- Cas extrême : fichier avec seulement nom + entreprise.
- Refuser l'import en mode strict OU n'autoriser que la création (jamais d'update).
- Avertissement explicite côté UI.

## 6. Données à dériver et stocker

Pour qu'une `stableIdentityKey` soit utilisable au scan en O(1), il faut :

- Stocker `stable_identity_key` sur `Registration` (`String`, nullable au début).
- Stocker la même `stable_identity_key` sur `QrCredential` au moment de l'émission.
- Index unique partiel : `(event_id, stable_identity_key) WHERE stable_identity_key IS NOT NULL`.
- Si plusieurs sources sont possibles, stocker la **source effective** utilisée (`external_reference`, `email`, etc.) pour debug.

> Implémentation à définir en Phase 2 de [11-future-implementation-plan-draft.md](./11-future-implementation-plan-draft.md). Aucune modification de schéma à ce stade.

## 7. Risques résiduels

- Backfill de `stable_identity_key` sur les registrations existantes : règle de dérivation à figer (probablement `external_reference` || `normalized_email`).
- Migration des QR existants : ils continuent d'encoder l'UUID `registration.id`, mais le scan doit aussi savoir les résoudre (rétro-compatibilité).
- Risque de fausses fusions si la règle de matching est trop laxiste (préférer trop conservateur, quitte à produire des `needs_review`).
