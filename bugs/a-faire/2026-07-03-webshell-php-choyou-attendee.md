# 🚨 Sécurité — fichiers PHP malveillants injectés dans `choyou.fr/_/attendee/`

> **Date signalement** : 2026-07-03 · **Signalé par** : équipe (matin)
> **Cible** : hébergement mutualisé `choyou.fr`, dossier `/_/attendee/` (et sous-dossiers)
> **Status** : 🔴 Ouvert — action requise de Rabie
> **⚠️ NE concerne PAS le VPS NestJS** (`51.75.252.74` / api.attendee.fr). C'est l'**ancien système PHP** sur le mutualisé.

## Message pour Rabie

Des fichiers PHP malveillants sont injectés **silencieusement** dans le dossier `/_/attendee/`
(et ses sous-dossiers) sur l'hébergement `choyou.fr`.

**À faire :**
1. Passer en revue l'**arborescence complète** de `/_/attendee/` (et sous-dossiers).
2. **Supprimer** les fichiers non désirés / non reconnus.

**Déjà fait aujourd'hui (03/07)** : 2 fichiers de ce type ont été supprimés.

## Contexte (ce qu'on sait)

- `/_/` sur `choyou.fr` = ancien socle PHP (mêmes chemins que `debug_token.php`,
  `staging_matinale/` référencés dans `attendee-ems-back/scripts/legacy/migration-data-from-ems-to-attendee/`).
- Un fichier PHP « injecté silencieusement » = très probablement un **webshell / backdoor**
  (porte dérobée permettant à un attaquant d'exécuter du code, uploader d'autres fichiers,
  spammer, ou pivoter). Le fait qu'ils **réapparaissent** après suppression indique en général
  une **persistance** : soit un accès toujours ouvert (FTP/SSH/CMS compromis), soit un script
  déjà présent qui régénère les fichiers (cron mutualisé, autre PHP infecté ailleurs sur le compte).

## Limite d'investigation

Pas d'accès à l'hébergement mutualisé `choyou.fr` depuis le workspace (l'agent n'a accès qu'au
VPS `51.75.252.74`). L'investigation fichier par fichier doit être faite par Rabie via FTP/SSH
du mutualisé OVH. Checklist ci-dessous.

## Checklist nettoyage + durcissement (pour Rabie)

**1. Constat / preuve**
- [ ] Lister par date de modif : `find /_/attendee -name '*.php' -newermt '2026-06-01' -printf '%TY-%Tm-%Td %p\n' | sort` (repérer les ajouts récents).
- [ ] Repérer les patterns de webshell : `grep -rlE 'eval\(|base64_decode|gzinflate|str_rot13|assert\(|preg_replace.*/e|system\(|shell_exec|passthru|\$_(GET|POST|REQUEST)\[' /_/attendee/`.
- [ ] Chercher aussi hors `attendee/` (l'infection touche souvent tout le compte mutualisé) : mêmes greps sur la racine `/_/` et le home.
- [ ] Garder une **copie** d'un fichier suspect avant suppression (analyse), ne pas juste effacer.

**2. Nettoyage**
- [ ] Supprimer les fichiers non reconnus.
- [ ] Vérifier les fichiers **légitimes modifiés** (injection en tête/pied de `index.php` etc.), pas seulement les nouveaux fichiers.
- [ ] Vérifier `.htaccess` (redirections/handlers PHP malveillants ajoutés).

**3. Persistance (sinon ça revient)**
- [ ] Vérifier les **crons** du compte mutualisé (panel OVH) — un cron peut re-déposer les fichiers.
- [ ] Changer **tous les mots de passe** : FTP/SFTP, SSH, comptes admin de l'ancien PHP, DB.
- [ ] Vérifier les comptes FTP secondaires créés à l'insu.
- [ ] Regarder les **logs d'accès** OVH pour l'IP/UA qui touche ces `.php`.

**4. Décision de fond**
- [ ] `/_/attendee/` est du **legacy** : s'il n'est plus utilisé en prod, envisager de le **retirer complètement** de l'hébergement (supprime la surface d'attaque). Confirmer d'abord qu'aucune landing / redirection active ne l'utilise.

## Sévérité

🔴 **Haute** — présence d'une backdoor active sur un hébergement du domaine `choyou.fr`.
Risque : défiguration, spam sortant (blacklist du domaine), fuite de données de l'ancienne base,
pivot vers d'autres sites du même compte mutualisé.
