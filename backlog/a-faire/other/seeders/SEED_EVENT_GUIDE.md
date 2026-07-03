# Guide - Seed pour l'événement spécifique

Ce guide explique comment remplir les données pour l'événement avec l'ID `8639f5cc-a4b5-4790-89a5-ffcb96f82c81`.

## 📋 Vue d'ensemble

Les nouveaux seeders créés permettent de :
1. **Créer des types de participants** (AttendeeType) : VIP, Conférencier, Sponsor, Presse, Participant, Staff
2. **Associer ces types à l'événement** (EventAttendeeType) avec des capacités définies
3. **Créer des inscriptions** (Registration) avec 20 participants variés

## 🚀 Utilisation rapide

### Option 1 : Script dédié (Recommandé)

Exécutez le script dédié qui fait tout automatiquement :

```bash
npm run db:seed:event
```

ou directement :

```bash
npx ts-node prisma/seeders/seed-specific-event.ts
```

### Option 2 : Seeders individuels

Vous pouvez aussi exécuter les seeders individuellement dans cet ordre :

```typescript
import { seedAttendeeTypes } from './prisma/seeders/attendee-types.seeder';
import { seedEventAttendeeTypes } from './prisma/seeders/event-attendee-types.seeder';
import { seedRegistrationsForEvent } from './prisma/seeders/registrations.seeder';

// 1. Créer les types de participants
await seedAttendeeTypes();

// 2. Associer les types à l'événement
await seedEventAttendeeTypes();

// 3. Créer les inscriptions
await seedRegistrationsForEvent();
```

## 📊 Données créées

### Types de participants (AttendeeType)

| Code | Nom | Couleur | Icône | Capacité par défaut |
|------|-----|---------|-------|---------------------|
| VIP | VIP | #FFD700 (Or) | star | 50 |
| SPEAKER | Conférencier | #9C27B0 (Violet) | microphone | 20 |
| SPONSOR | Sponsor | #FF9800 (Orange) | briefcase | 30 |
| PRESS | Presse | #2196F3 (Bleu) | camera | 25 |
| PARTICIPANT | Participant | #4CAF50 (Vert) | user | 500 |
| STAFF | Staff | #607D8B (Gris) | users | 40 |

### Inscriptions (Registration)

Le seeder crée **20 inscriptions** avec :
- **Répartition des statuts** :
  - 80% approuvées (`approved`)
  - 10% en attente (`awaiting`)
  - 5% refusées (`refused`)
  - 5% annulées (`cancelled`)

- **Types de participation** :
  - Distribution aléatoire entre : `onsite`, `online`, `hybrid`

- **Types de participants** :
  - Distribution cyclique entre tous les types disponibles

- **Participants** :
  - 20 participants avec des profils variés (développeurs, managers, journalistes, etc.)
  - Données complètes : nom, prénom, email, entreprise, fonction, pays

## 🔍 Vérification

Après l'exécution du seed, vous pouvez vérifier les données :

### Via Prisma Studio
```bash
npm run db:studio
```

Puis naviguez vers :
- `attendee_types` : Vérifier les 6 types créés
- `event_attendee_types` : Vérifier l'association avec l'événement
- `registrations` : Vérifier les 20 inscriptions

### Via requête SQL
```sql
-- Vérifier l'événement
SELECT * FROM events WHERE id = '8639f5cc-a4b5-4790-89a5-ffcb96f82c81';

-- Vérifier les types associés
SELECT eat.*, at.name, at.code 
FROM event_attendee_types eat
JOIN attendee_types at ON eat.attendee_type_id = at.id
WHERE eat.event_id = '8639f5cc-a4b5-4790-89a5-ffcb96f82c81';

-- Vérifier les inscriptions
SELECT r.*, a.first_name, a.last_name, a.email, at.name as attendee_type
FROM registrations r
JOIN attendees a ON r.attendee_id = a.id
LEFT JOIN event_attendee_types eat ON r.event_attendee_type_id = eat.id
LEFT JOIN attendee_types at ON eat.attendee_type_id = at.id
WHERE r.event_id = '8639f5cc-a4b5-4790-89a5-ffcb96f82c81';
```

## ⚠️ Notes importantes

1. **Idempotence** : Les seeders sont idempotents. Si vous les exécutez plusieurs fois, ils ne créeront pas de doublons.

2. **Prérequis** : L'événement avec l'ID `8639f5cc-a4b5-4790-89a5-ffcb96f82c81` doit exister dans la base de données.

3. **Organisation** : Les seeders utilisent l'organisation `acme-corp` par défaut. Assurez-vous qu'elle existe.

4. **Ordre d'exécution** : Respectez l'ordre des seeders pour éviter les erreurs de contraintes de clés étrangères.

## 🛠️ Personnalisation

### Modifier les types de participants

Éditez le fichier `prisma/seeders/attendee-types.seeder.ts` :

```typescript
const attendeeTypesData = [
  {
    org_id: acmeOrg.id,
    code: 'CUSTOM_TYPE',
    name: 'Mon Type Personnalisé',
    color_hex: '#FF0000',
    text_color_hex: '#FFFFFF',
    icon: 'custom-icon',
    is_active: true,
    sort_order: 7,
  },
  // ...
];
```

### Modifier les capacités par type

Éditez le fichier `prisma/seeders/event-attendee-types.seeder.ts` dans la section `switch` :

```typescript
switch (attendeeType.code) {
  case 'VIP':
    capacity = 100; // Augmenter la capacité VIP
    break;
  // ...
}
```

### Modifier le nombre d'inscriptions

Éditez le fichier `prisma/seeders/registrations.seeder.ts` et ajoutez/supprimez des entrées dans `attendeesData`.

## 📝 Fichiers créés

- `prisma/seeders/attendee-types.seeder.ts` - Création des types de participants
- `prisma/seeders/event-attendee-types.seeder.ts` - Association types ↔ événement
- `prisma/seeders/registrations.seeder.ts` - Création des inscriptions
- `prisma/seeders/seed-specific-event.ts` - Script orchestrateur
- `prisma/seeders/SEED_EVENT_GUIDE.md` - Ce guide

## 🐛 Dépannage

### Erreur : "Event not found"
L'événement avec l'ID spécifié n'existe pas. Vérifiez l'ID ou créez l'événement d'abord.

### Erreur : "Organization not found"
L'organisation `acme-corp` n'existe pas. Exécutez d'abord le seed principal :
```bash
npm run db:seed
```

### Erreur : "Unique constraint violation"
Les données existent déjà. C'est normal si vous ré-exécutez le seed. Le script détecte les doublons et les ignore.

## 📞 Support

Pour toute question ou problème, consultez :
- Le README principal : `prisma/seeders/README.md`
- La documentation Prisma : https://www.prisma.io/docs
