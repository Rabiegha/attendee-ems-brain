# Chantier G — Wallet Apple + Google

> Billet en Wallet Apple + Google. Harnais dans B · onboarding stores à lancer tôt (calendaire) ·
> Google fast-follow · Apple hors chemin critique.

- **Plan maître :** [../00-plan-action.md](../00-plan-action.md) (ligne G du tableau)
- **Avancement (%) :** [../03-suivi-chantiers.md](../03-suivi-chantiers.md) (ligne **G**)
- **Statut :** 🟡 V2 — onboarding stores seul à lancer (calendaire)

---

## Analyse Codex — lien avec le badge maker Attendee (13/07)

Question : est-ce que les fichiers Wallet Apple / Google peuvent suivre le même flux que les PDF
de badge, ou être générés facilement depuis les templates du badge maker ?

### Verdict court

Non, **un Wallet n'est pas un PDF** et ne peut pas être généré tel quel depuis le layout libre du
badge maker. En revanche, le badge maker et le moteur de génération de badges peuvent fournir une
bonne partie des **données métier** nécessaires au Wallet.

```txt
Badge/PDF :
template_data -> HTML/CSS -> Puppeteer -> PDF/image

Wallet Apple :
pass.json + images imposées + barcode + signature Apple -> .pkpass

Google Wallet :
objet Wallet structuré + lien/JWT signé ou API Google -> "Add to Google Wallet"
```

### Ce que le badge maker fait aujourd'hui

Le badge maker est un éditeur visuel libre :

- éléments `text`, `image`, `qrcode`, `group` ;
- positions `x/y`, tailles, rotations, styles ;
- dimensions en mm/px ;
- variantes conditionnelles ;
- variables `{{firstName}}`, `{{lastName}}`, `{{eventName}}`, `{{registrationId}}`,
  `{{sessionChoice}}`, etc. ;
- rendu back en HTML/CSS puis PDF/image via Puppeteer.

Références code :

```txt
attendee-ems-front/src/pages/BadgeDesigner/*
attendee-ems-front/src/shared/types/badge.types.ts
attendee-ems-back/src/modules/badge-generation/badge-generation.service.ts
attendee-ems-back/src/modules/badge-templates/interfaces/badge-design.interface.ts
```

### Ce qu'on peut réutiliser pour Wallet

À réutiliser :

- les données `registration` / `attendee` / `event` ;
- la logique de sélection de template selon event/type participant, si utile ;
- les variables déjà normalisées : `firstName`, `lastName`, `fullName`, `company`,
  `jobTitle`, `eventName`, `sessionChoice`, etc. ;
- le QR signé via `QrTokenService.sign(registration.id)` ;
- éventuellement les logos/couleurs/images du badge comme source de branding ;
- éventuellement une image générée du badge comme preview/thumbnail secondaire.

### Ce qu'on ne peut pas réutiliser tel quel

À ne pas supposer réutilisable automatiquement :

- le layout HTML/CSS complet ;
- les coordonnées `x/y` du badge ;
- les rotations, groupes, calques, `zIndex` ;
- la sortie PDF/image ;
- Puppeteer comme moteur principal Wallet.

Raison : Apple Wallet et Google Wallet imposent des modèles structurés. Ils ne rendent pas une page
HTML libre comme un PDF. Il faut alimenter des champs typés :

```txt
primaryFields
secondaryFields
auxiliaryFields
barcode
logo
backgroundColor
foregroundColor
labelColor
```

### Recommandation V1 / event

Pour LFD, ne pas essayer de convertir automatiquement un badge template en Wallet.

Faire plutôt un générateur Wallet minimal structuré :

```txt
registration + event + QR signé
-> WalletService
-> Apple .pkpass
-> Google Wallet save link
```

Configuration minimale par event :

- logo ;
- couleur principale ;
- nom de l'événement ;
- lieu/date ;
- champs à afficher ;
- payload du barcode/QR ;
- éventuellement lien vers le billet PDF existant.

### Recommandation V2

Après event, connecter progressivement Wallet au badge maker :

- proposer un preset Wallet à partir du badge template ;
- récupérer logo/couleurs/background depuis le badge ;
- exposer un mini-mapping UI : `champ Wallet -> variable Attendee` ;
- générer une image preview depuis le badge si utile ;
- garder Apple/Google comme formats structurés séparés, pas comme "export PDF".

Conclusion :

```txt
Le badge maker peut alimenter Wallet en données et branding.
Il ne doit pas être considéré comme un moteur de rendu Wallet.
```
