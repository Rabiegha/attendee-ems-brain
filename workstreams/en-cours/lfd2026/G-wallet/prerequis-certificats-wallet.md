# Prerequis certificats Wallet Apple + Google

> Objet : cadrer ce qu'il faut lancer tot pour pouvoir generer des billets Wallet
> apres l'event / en V2, sans bloquer le perimetre fin juillet.
>
> Statut recommande : **onboarding consoles a lancer des maintenant**. Le dev peut rester post-event,
> mais les validations compte/certificats sont calendaires.

---

## Verdict court

Pour LFD, le Wallet n'est pas un export du PDF badge. C'est un format structure, signe, publie via
les ecosystemes Apple et Google.

- **Apple Wallet** : besoin d'un compte Apple Developer organisation, d'un `Pass Type ID`, d'un
  certificat de signature de pass, et du stockage securise de la cle privee.
- **Google Wallet** : pas de certificat Apple-like ; besoin d'un compte Google Wallet API Issuer,
  d'un projet Google Cloud, d'un service account, d'une classe/objet Wallet et d'un acces
  publication hors mode demo.
- **Risque principal** : les comptes et validations ne dependent pas du code. A lancer tot.
- **Recommandation V1** : QR code signe Attendee + donnees inscription/event + branding minimal.
  Pas de NFC, pas de conversion automatique du badge maker.

---

## Sources officielles a garder sous la main

- Apple Wallet developer overview : https://developer.apple.com/wallet/
- Apple Wallet passes documentation : https://developer.apple.com/documentation/walletpasses
- Apple Account - certificats / identifiers : https://developer.apple.com/account/resources/
- Google Wallet onboarding guide : https://developers.google.com/wallet/generic/getting-started/onboarding-guide
- Google Wallet issuer account : https://developers.google.com/wallet/generic/getting-started/issuer-onboarding
- Google Wallet publishing access : https://developers.google.com/wallet/generic/test-and-go-live/request-publishing-access
- Google Wallet event tickets : https://developers.google.com/wallet/tickets/events
- Google Wallet issue via web/email/SMS : https://developers.google.com/wallet/generic/web

Note 13/07/2026 : les pages Google indiquent que les nouveaux issuer accounts demarrent en
**Demo Mode** ; la publication publique necessite une demande d'acces publication.

---

## Apple Wallet

### Ce qu'il faut obtenir

| Element | Requis | Responsable probable | Commentaire |
| --- | --- | --- | --- |
| Apple Developer Program | Oui | Attendee / client proprietaire du compte | Idealement un compte organisation, pas un Apple ID personnel. |
| Acces `Account Holder` ou `Admin` | Oui | Attendee | Necessaire pour creer identifiers/certificats. |
| `Pass Type ID` | Oui | Attendee / dev avec acces | Format habituel : `pass.fr.attendee.lfd2026` ou `pass.fr.attendee.ticket`. |
| Certificat `Pass Type ID Certificate` | Oui | Dev/ops | Sert a signer les `.pkpass`. |
| Cle privee associee au certificat | Oui | Ops | A generer et stocker comme secret. Si perdue, il faut recreer/renouveler. |
| Certificat Apple WWDR/intermediaire | Oui selon librairie | Dev/ops | Souvent necessaire dans la chaine de signature du `.pkpass`. |
| Assets Apple | Oui | Design / event | Icones, logo, couleurs, event name, lieu/date. |
| Endpoint HTTPS public | Oui | Back Attendee | Pour telechargement `.pkpass`; obligatoire si updates web service. |

### Etapes Apple

1. Verifier l'existence et le proprietaire du compte Apple Developer.
2. Obtenir un acces admin au portail Apple Developer.
3. Creer un `Pass Type ID` dedie.
4. Generer une CSR depuis une machine de confiance.
5. Creer le certificat `Pass Type ID Certificate` depuis Apple Developer.
6. Exporter le certificat + cle privee en `.p12`, avec mot de passe fort.
7. Stocker les secrets dans l'infra cible, jamais dans Git :
   - `APPLE_PASS_TYPE_ID`
   - `APPLE_TEAM_ID`
   - `APPLE_PASS_CERT_P12`
   - `APPLE_PASS_CERT_PASSWORD`
   - certificat intermediaire Apple si la lib le demande
8. Generer un premier `.pkpass` de test :
   - `pass.json`
   - assets imposes
   - `manifest.json`
   - signature
   - archive `.pkpass`
9. Tester l'ouverture sur iPhone reel via lien HTTPS, email ou AirDrop.
10. Ajouter les controles prod :
    - MIME type `application/vnd.apple.pkpass`
    - nom de fichier stable
    - logs generation
    - rotation certificat
    - fallback billet PDF si echec Wallet.

### Contraintes techniques Apple

- Apple Wallet rend des champs structures, pas du HTML/CSS.
- Le QR/barcode doit reprendre le token signe Attendee deja utilise pour le scan.
- Chaque pass doit avoir un `serialNumber` stable et unique.
- Un pass peut etre statique. Les updates push/web service sont optionnels pour une V1.
- Les images doivent respecter les contraintes Wallet ; le rendu peut varier selon device.
- Tester sur iPhone reel est indispensable. Le simulateur et l'aperçu ne suffisent pas.

### Delais Apple

| Scenario | Delai prudent |
| --- | --- |
| Compte Apple Developer deja actif + acces admin disponible | 0,5 a 1 j |
| Creation Pass Type ID + certificat + premier `.pkpass` signe | quelques heures a 1 j |
| Integration back V1 apres certificats | 1 a 2 j |
| Probleme acces compte, 2FA, role insuffisant, compte perso au lieu d'organisation | 2 a 7 j+ |
| Renouvellement/rotation certificat oublie | incident a date d'expiration |

Apple n'a pas d'equivalent "publishing access" pour distribuer un simple `.pkpass` signe. Le
goulet est surtout l'acces au compte, la signature correcte, et les tests sur device.

---

## Google Wallet

### Ce qu'il faut obtenir

| Element | Requis | Responsable probable | Commentaire |
| --- | --- | --- | --- |
| Compte Google proprietaire | Oui | Attendee | Eviter le Gmail personnel d'un dev. |
| Google Wallet API Issuer account | Oui | Attendee / ops | Cree dans Google Pay & Wallet Console. |
| Business Profile complete | Oui pour publication | Attendee | Sert a verifier l'organisation. |
| Projet Google Cloud | Oui | Ops | API Google Wallet activee. |
| Service account | Oui | Ops/dev | Auth REST API + signature JWT. |
| Cle service account | Oui si flow JWT serveur | Ops | Secret critique, rotation a prevoir. |
| Passes Class | Oui | Dev/ops | Template partage : event ticket LFD. |
| Passes Object | Oui | Back | Objet individuel par inscrit/billet. |
| Publishing access | Oui pour public | Attendee / ops | Sans ca, passes limites au mode demo/test. |

### Etapes Google

1. Creer ou identifier le compte Google proprietaire.
2. Creer le Google Wallet API Issuer account dans Google Pay & Wallet Console.
3. Completer le Business Profile et le profil de paiement si demande.
4. Creer/choisir le projet Google Cloud.
5. Activer Google Wallet API.
6. Creer un service account dedie.
7. Ajouter le service account a l'issuer account avec les droits necessaires.
8. Creer une premiere `EventTicketClass` ou, en fallback, une `GenericClass`.
9. Creer un objet de test pour une inscription LFD.
10. Generer un lien `Add to Google Wallet` :
    - class/object
    - JWT signe avec la cle service account
    - URL `https://pay.google.com/gp/v/save/<signed_jwt>`
11. Tester sur compte Google de test.
12. Demander le publishing access.
13. Re-tester apres validation : disparition du marqueur `[TEST ONLY]`, ajout par un compte non test.

### Event ticket ou generic pass ?

Option recommandee pour LFD : **Event Tickets**.

Avantages :

- semantique adaptee a l'entree evenement ;
- champs date/lieu/event plus naturels ;
- experience Google Wallet plus attendue pour un billet.

Fallback possible : **Generic Pass** si l'event ticket demande trop de parametrage ou si la V1 doit
aller tres vite. Ca reste acceptable pour un badge/billet simple, mais moins propre produit.

### Contraintes techniques Google

- En mode demo, les passes sont limites aux admins/devs/test accounts et affichent `[TEST ONLY]`.
- La publication publique passe par une revue Google.
- Le lien web/email/SMS contient un JWT signe.
- Google recommande un JWT court ; au-dela d'environ 1800 caracteres, certains navigateurs peuvent
  tronquer le lien. Pour des objets riches, creer class/object via API puis mettre seulement les
  references dans le JWT.
- L'objet Wallet doit avoir un identifiant stable. Prevoir convention :
  `issuerId.registration.<registrationId>` ou equivalent.
- Le QR/barcode doit rester coherent avec le scan Attendee existant.
- Tester sur Android reel avec Google Wallet installe, et sur un compte hors test apres publication.

### Delais Google

| Scenario | Delai prudent |
| --- | --- |
| Issuer account + projet GCP + service account | 0,5 a 1 j |
| Business Profile / profil paiement | 0,5 a 2 j selon acces administratifs |
| Premier pass demo ajoute a Google Wallet | 0,5 a 1 j apres credentials |
| Integration back V1 apres credentials | 1 a 3 j |
| Publishing access Google | 1 a 5 j ouvrés prudent, sans SLA a supposer |
| Blocage compte/profil/verif organisation | 1 a 2 semaines possible |

Google est donc le vrai sujet calendaire : a lancer avant le dev complet.

---

## Secrets et securite

Secrets a prevoir :

```txt
APPLE_TEAM_ID
APPLE_PASS_TYPE_ID
APPLE_PASS_CERT_P12
APPLE_PASS_CERT_PASSWORD
APPLE_WWDR_CERT

GOOGLE_WALLET_ISSUER_ID
GOOGLE_WALLET_SERVICE_ACCOUNT_EMAIL
GOOGLE_WALLET_SERVICE_ACCOUNT_KEY_JSON
GOOGLE_WALLET_CLASS_ID
```

Regles :

- jamais de certificat, `.p12`, `.pem`, `.key` ou JSON service account dans Git ;
- stockage dans secrets manager / variables CI/CD chiffrees ;
- acces limite aux personnes qui deployent le back ;
- rotation documentee avant expiration ;
- revoke/rotation immediate si partage accidentel ;
- separer environnement test/staging/prod si possible ;
- journaliser la generation Wallet sans logger le payload secret ni le token brut si sensible.

---

## Donnees Attendee necessaires

Minimum V1 :

| Donnee | Source Attendee | Usage Wallet |
| --- | --- | --- |
| `registration.id` | Back | serial/object id, QR signe |
| `QrTokenService.sign(registration.id)` | Back | barcode/QR |
| `firstName`, `lastName`, `fullName` | Registration/attendee | nom affiche |
| `company`, `jobTitle` | Attendee | champs secondaires si disponibles |
| `eventName` | Event | titre pass |
| date/debut/fin event | Event | pertinence Wallet |
| lieu/adresse | Event | affichage + pertinence |
| type participant / ticket | Registration | label billet |
| logo/couleur event | Branding | rendu Apple/Google |
| lien billet PDF | Back/front | fallback ou lien secondaire |

Hors V1 :

- conversion automatique du layout badge maker ;
- rendu PDF dans Wallet ;
- NFC / Smart Tap ;
- push notifications Wallet ;
- updates dynamiques complexes ;
- grouping multi-billets.

---

## Plan de lancement recommande

### Maintenant

- Identifier les proprietaires des comptes Apple et Google.
- Confirmer que les comptes sont organisationnels et recuperables.
- Lancer Apple Developer access + Pass Type ID.
- Lancer Google Wallet Issuer + Business Profile + projet GCP.
- Obtenir les assets minimaux LFD : logo lisible, couleur principale, nom exact, dates, lieu.

### Quand les acces sont disponibles

- Generer Apple `.p12` et documenter expiration/rotation.
- Creer Google service account + class de test.
- Produire un pass Apple et un pass Google demo sur une inscription de test.
- Tester sur iPhone et Android reels.

### Avant publication publique

- Valider QR scan bout en bout depuis Wallet.
- Verifier fallback PDF si Wallet indisponible.
- Demander publishing access Google.
- Tester un compte Google non test apres validation.
- Ecrire runbook support : "le bouton Wallet ne marche pas", "pass affiche TEST ONLY",
  "QR invalide", "certificat expire", "lien tronque".

---

## Risques

| Risque | Impact | Mitigation |
| --- | --- | --- |
| Acces compte Apple/Google chez une seule personne | Blocage calendaire | Comptes organisationnels + 2 admins minimum. |
| Apple cert genere sur machine non maitrisee | Secret fragile/perdu | CSR sur machine ops, export `.p12`, coffre a secrets. |
| Certificat Apple expire non surveille | Tous les nouveaux `.pkpass` echouent | Date expiration dans runbook + alerte calendrier. |
| Google reste en Demo Mode | Passes publics marques/test ou impossibles | Demande publishing access tot, test compte hors allowlist. |
| Business Profile Google incomplet | Review bloquee | Attendee doit fournir infos legales, adresse, paiement si demande. |
| JWT Google trop long | Bouton "Add" instable | Creer class/object via API, JWT minimal. |
| QR Wallet diverge du QR PDF | Scan event casse | Utiliser exactement le meme service de signature QR. |
| Pass partage entre personnes | Fraude entree | QR signe + etat serveur + invalidation apres scan si regle event. |
| Timezone/date mal affichee | Mauvaise experience participant | Dates ISO + timezone Europe/Paris explicite. |
| Assets non conformes | Rendu cheap ou rejet implicite | Preview devices + assets dedies Wallet. |
| Fallback absent | Support massif le jour J | PDF reste source de verite V1. |

---

## Decision produit V1

Pour LFD, viser :

```txt
registration + event + QR signe
-> Apple .pkpass telechargeable
-> Google Add to Wallet link
-> fallback PDF existant
```

Critere de succes V1 :

- un inscrit peut ajouter son billet a Apple Wallet depuis iPhone ;
- un inscrit peut ajouter son billet a Google Wallet depuis Android ;
- le QR Wallet est scannable par le flux Attendee existant ;
- si Wallet echoue, le PDF continue de fonctionner ;
- aucune operation manuelle n'est requise par billet.

Conclusion : **le chantier technique peut attendre V2, mais l'onboarding Apple/Google ne doit pas
attendre le debut du dev**.
