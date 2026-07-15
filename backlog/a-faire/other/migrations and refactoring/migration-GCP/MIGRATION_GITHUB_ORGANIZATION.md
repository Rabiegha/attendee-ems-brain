# Migration vers une Organisation GitHub

> **Date** : 2 avril 2026  
> **Contexte** : Repos actuels sous le compte personnel `Rabiegha`  
> **Objectif** : Migrer vers une organisation GitHub pour une meilleure gestion des accès
> **Décision 15/07/2026** : **report post-event LFD 2026**. Pendant le rush, ne pas prendre GitHub Pro
> sur le compte perso et ne pas transférer les repos. Reprendre après l'event avec une migration propre
> vers une organisation Attendee.

## Statut

🔵 **À faire post-event**.

Raison du report : la migration touche l'ownership GitHub, les remotes, les permissions, les secrets
GitHub Actions, GHCR et potentiellement la chaîne CD. Même si GitHub conserve des redirections, ce n'est
pas un changement à introduire pendant la fenêtre de stabilisation LFD 2026. Pour l'event, on garde les
repos sous `Rabiegha` et la chaîne CI/CD validée telle quelle.

---

## Pourquoi migrer vers une organisation ?

### Problèmes actuels (compte personnel)

- Les repos sont liés au compte `Rabiegha` directement
- Pour donner accès à quelqu'un → ajout en collaborateur sur **chaque** repo individuellement
- Pas de gestion par équipe (dev, ops, design, etc.)
- Pas de contrôle granulaire (lecture seule vs écriture par repo)
- Si le compte personnel est compromis, tous les repos le sont

### Avantages d'une organisation

- **Gestion centralisée** : un seul endroit pour gérer tous les accès
- **Équipes (Teams)** : créer des groupes avec des permissions par repo
- **Rôles granulaires** : `Read`, `Triage`, `Write`, `Maintain`, `Admin` par repo et par équipe
- **Audit log** : traçabilité de qui a fait quoi
- **Facturation centralisée** : si besoin de passer en GitHub Team plus tard
- **GitHub Actions** : secrets d'organisation partagés entre repos
- **Sécurité** : 2FA obligatoire pour les membres, policies de sécurité

---

## Plan de migration

### Étape 1 — Créer l'organisation

1. Aller sur https://github.com/organizations/plan
2. Choisir **Free** (suffisant pour repos privés illimités + équipes de base)
3. Nom suggéré : `attendee-ems` ou `attendee-io`
4. Ajouter le compte `Rabiegha` comme **Owner**

### Étape 2 — Transférer les repos

Pour chaque repo, depuis le compte `Rabiegha` :

1. **Settings** → **General** → scroll jusqu'à **Danger Zone**
2. **Transfer ownership** → Sélectionner la nouvelle organisation
3. Confirmer le nom du repo

**Repos à transférer** :

| Repo | Branche par défaut |
|---|---|
| `attendee-ems-back` | `main` |
| `attendee-ems-front` | `main` |
| `attendee-ems-mobile` | `main` |
| `attendee-ems-print-client` | `feature/print-queue` |
| `attendee-context-hub` | `main` |

> **Note** : GitHub crée automatiquement des redirections depuis l'ancien URL (`Rabiegha/repo`) vers le nouveau (`org/repo`). Les remotes git existants continuent de fonctionner grâce à la redirection, mais il est recommandé de les mettre à jour.

### Étape 3 — Mettre à jour les remotes locaux

Après le transfert, sur chaque machine de développement :

```bash
# Pour chaque repo, mettre à jour l'URL remote
cd attendee-ems-back
git remote set-url origin git@github.com:NOUVELLE_ORG/attendee-ems-back.git

cd ../attendee-ems-front
git remote set-url origin git@github.com:NOUVELLE_ORG/attendee-ems-front.git

cd ../attendee-ems-mobile
git remote set-url origin git@github.com:NOUVELLE_ORG/attendee-ems-mobile.git

cd ../attendee-ems-print-client
git remote set-url origin git@github.com:NOUVELLE_ORG/attendee-ems-print-client.git

cd ../attendee-context-hub
git remote set-url origin git@github.com:NOUVELLE_ORG/attendee-context-hub.git
```

### Étape 4 — Créer les équipes

Aller dans **Organization → Teams** et créer les équipes nécessaires :

| Équipe | Permission par défaut | Repos |
|---|---|---|
| `core-dev` | **Write** | Tous les repos |
| `frontend` | **Write** sur front, **Read** sur back | `ems-front`, `ems-mobile` |
| `backend` | **Write** sur back, **Read** sur front | `ems-back` |
| `ops` | **Maintain** | Tous les repos |
| `externe` | **Read** | Repos sélectionnés |

### Étape 5 — Configurer la sécurité

1. **Organization → Settings → Authentication security**
   - Activer **Require 2FA** pour tous les membres

2. **Organization → Settings → Member privileges**
   - Base permissions : `None` (pas d'accès par défaut, tout passe par les Teams)
   - Désactiver la création de repos par les membres (sauf admins)
   - Désactiver le fork de repos privés

3. **Par repo → Settings → Branches**
   - Protéger la branche `main` :
     - Require pull request reviews
     - Require status checks (CI)
     - Disable force push

### Étape 6 — Mettre à jour les services connectés

Vérifier et mettre à jour :

- [ ] **CI/CD** (GitHub Actions) : les workflows continuent de fonctionner si les secrets sont au niveau org
- [ ] **Cloud Run / GCP** : mettre à jour les URLs de repos dans les configs de build
- [ ] **Vercel / Netlify** (si utilisé pour le front) : reconnecter le repo
- [ ] **Expo / EAS** (mobile) : mettre à jour la config dans `eas.json`
- [ ] **Deploy scripts** : vérifier `deploy-back.sh`, `deploy-front.sh`
- [ ] **Docker images** : si les tags référencent le nom du repo

---

## Donner accès à un externe (sans l'ajouter à l'org)

### Option A — Outside Collaborator (recommandé)

```
Organization → Repo → Settings → Collaborators → Add people
→ Sélectionner "Outside collaborator"
→ Choisir le rôle : Read / Triage / Write
```

- L'externe voit **uniquement** ce repo, pas les autres
- Pas de visibilité sur l'organisation
- Révocable à tout moment

### Option B — Équipe "externe" avec accès limité

1. Créer une Team `externe`
2. Ajouter uniquement les repos autorisés avec permission `Read`
3. Inviter la personne dans cette Team

### Option C — GitHub Fine-grained Personal Access Token

Pour un accès programmatique (ex: CI externe, audit) :

1. **Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Créer un token scopé à un seul repo
3. Permissions : `Contents: Read-only`
4. Expiration : 30 jours max
5. Partager le token de façon sécurisée

### Option D — Partage ponctuel sans accès GitHub

Si la personne n'a pas besoin d'un accès permanent :

- **Gist secret** : copier le contenu dans un gist privé, partager le lien
- **Export ZIP** : `git archive --format=zip HEAD > export.zip`
- **Google Drive / Notion** : pour de la documentation

---

## Comparaison des plans GitHub

| Fonctionnalité | Free | Team (4$/user/mois) |
|---|---|---|
| Repos privés illimités | ✅ | ✅ |
| Équipes de base | ✅ | ✅ |
| Outside collaborators | ✅ | ✅ |
| Protected branches | ✅ | ✅ |
| Required reviewers | ✅ | ✅ |
| CODEOWNERS | ❌ | ✅ |
| Draft PRs | ✅ | ✅ |
| Environments (deploy) | ❌ | ✅ |
| Audit log API | ❌ | ✅ |
| SAML SSO | ❌ | ❌ (Enterprise) |

**Recommandation** : commencer avec **Free**, passer en **Team** quand l'équipe dépasse 3-4 personnes ou quand tu as besoin de CODEOWNERS / environments.

---

## Checklist post-migration

- [ ] Organisation créée
- [ ] Tous les repos transférés
- [ ] Remotes git mis à jour sur toutes les machines
- [ ] Équipes créées et membres assignés
- [ ] 2FA activé pour tous les membres
- [ ] Branches protégées configurées
- [ ] Base permissions réglé sur `None`
- [ ] CI/CD vérifié et fonctionnel
- [ ] Deploy scripts mis à jour
- [ ] Anciens accès collaborateurs nettoyés
