# Monitoring serveur — Netdata

Guide rapide pour accéder et administrer le monitoring temps réel du VPS de production (`51.75.252.74`).

---

## 1. Accès au dashboard

### Option A — Via le navigateur (recommandé, accès rapide)

URL : **https://api.attendee.fr/netdata/**

Authentification (basic auth) :
- **Utilisateur** : `admin`
- **Mot de passe** : celui défini sur le serveur (voir section « Modifier le mot de passe »)

> Le dashboard est proxifié par nginx (reverse proxy) vers l'instance Netdata locale qui tourne sur le port `19999` du VPS. Il n'est **pas** exposé directement sur Internet.

### Option B — Via Netdata Cloud (multi-serveurs, alertes, historique long)

URL : **https://app.netdata.cloud**

Le VPS (`vps-7451d3c3`) est *claimé* (connecté) au compte Netdata Cloud. Connexion avec les identifiants du compte cloud.

Pour **inviter un collègue** : `app.netdata.cloud` → menu de gauche → **Members / Invite Users**.
> ⚠️ L'invitation ne se fait **pas** depuis `api.attendee.fr/netdata/` (dashboard local) — uniquement depuis `app.netdata.cloud`.

### Option C — Tunnel SSH (sans passer par nginx)

```bash
ssh -L 19999:localhost:19999 debian@51.75.252.74
```
Puis ouvrir **http://localhost:19999** dans le navigateur (pas d'auth nginx, accès direct local).

---

## 2. Modifier le mot de passe (basic auth nginx)

Le fichier d'authentification est sur le VPS : `/opt/ems-attendee/backend/nginx/.htpasswd`

Se connecter au serveur puis lancer (remplacer `NOUVEAU_MDP` par le mot de passe voulu) :

```bash
ssh debian@51.75.252.74
sudo sh -c 'HASH=$(openssl passwd -apr1 NOUVEAU_MDP) && echo "admin:$HASH" > /opt/ems-attendee/backend/nginx/.htpasswd && docker exec ems-nginx nginx -s reload && echo OK'
```

> `htpasswd` est aussi disponible (`/usr/sbin/htpasswd`), mais `openssl passwd -apr1` évite toute dépendance et fonctionne partout.

Pour **ajouter** un second utilisateur (sans écraser le premier), retirer le `>` qui écrase et utiliser `>>` :

```bash
sudo sh -c 'HASH=$(openssl passwd -apr1 MDP_USER2) && echo "user2:$HASH" >> /opt/ems-attendee/backend/nginx/.htpasswd && docker exec ems-nginx nginx -s reload'
```

### Vérifier que le changement a pris

```bash
# 401 sans auth (bloqué) — attendu
curl -s -o /dev/null -w "%{http_code}\n" https://api.attendee.fr/netdata/

# 200 avec les bons identifiants — attendu
curl -s -o /dev/null -w "%{http_code}\n" -u admin:NOUVEAU_MDP https://api.attendee.fr/netdata/
```

---

## 3. Administration du service Netdata

Sur le VPS (`ssh debian@51.75.252.74`) :

```bash
# Statut
systemctl is-active netdata
systemctl status netdata --no-pager

# Redémarrer / arrêter / démarrer
sudo systemctl restart netdata
sudo systemctl stop netdata
sudo systemctl start netdata

# Vérifier qu'il écoute bien sur le port 19999
ss -tlnp | grep 19999

# Logs
sudo journalctl -u netdata -n 50 --no-pager
```

### Vérifier que le port direct n'est pas public

Depuis une machine externe :

```bash
curl -sS --connect-timeout 5 -o /dev/null -w "%{http_code}\n" http://51.75.252.74:19999/
```

Résultat attendu : échec de connexion / `000`. L'accès public doit passer uniquement par
`https://api.attendee.fr/netdata/` avec basic auth.

Sur le VPS, Netdata doit écouter seulement en local et sur le bridge Docker :

```bash
sudo ss -tlnp | grep 19999
# attendu : 127.0.0.1:19999 et 172.17.0.1:19999, pas 0.0.0.0:19999
```

La configuration attendue est versionnée dans le backend :

```bash
cd /opt/ems-attendee/backend
./monitoring/apply-netdata-system-bind.sh --check
```

Pour réappliquer la configuration après réinstallation/update Netdata :

```bash
cd /opt/ems-attendee/backend
./monitoring/apply-netdata-system-bind.sh
```

### Alertes EMS et webhook

Les alertes EMS sont installées dans le Netdata système :

```bash
cd /opt/ems-attendee/backend
./monitoring/apply-netdata-system-health.sh --check
```

Alarmes attendues :

- `ems_disk_usage`
- `ems_disk_usage_80`
- `ems_cpu_sustained`
- `ems_ram_usage`

Le canal d'alerte doit être posé sur le VPS, jamais dans Git :

```bash
cd /opt/ems-attendee/backend
echo 'NETDATA_WEBHOOK_URL=<webhook>' > .env.monitoring
chmod 600 .env.monitoring
./monitoring/apply-netdata-system-health.sh
```

Le même fichier est aussi lu par `monitoring/check-bullmq-queues.sh`.

Canal actuel : Discord `EMS Monitoring`.

Test validé le 2026-07-16 : alarme temporaire `ems_disk_webhook_test` sur `disk_space./`, passée en
`WARNING`, puis supprimée et Netdata redémarré.

### BullMQ queue check

Le VPS n'a pas `crontab`; la surveillance `/health/queues` utilise donc un timer systemd :

```bash
systemctl list-timers --all ems-bullmq-check.timer --no-pager
sudo systemctl status ems-bullmq-check.service --no-pager -n 20
sudo tail -n 50 /var/log/ems-bullmq-check.log
```

Le timer exécute toutes les 5 minutes :

```bash
/opt/ems-attendee/backend/monitoring/check-bullmq-queues.sh
```

---

## 4. Configuration technique (référence)

| Élément | Valeur |
|---|---|
| Version Netdata | `v2.10.3` |
| Port local | `19999` (host VPS, hors Docker) |
| Configuration | `/opt/netdata/etc/netdata/netdata.conf` (`[web] bind to = 127.0.0.1:19999 172.17.0.1:19999`) |
| Script de garde-fou | `monitoring/apply-netdata-system-bind.sh` |
| Script alertes EMS | `monitoring/apply-netdata-system-health.sh` |
| Canal webhook | `/opt/ems-attendee/backend/.env.monitoring` (`NETDATA_WEBHOOK_URL=...`) |
| Canal actuel | Discord `EMS Monitoring` |
| BullMQ timer | `ems-bullmq-check.timer` → `ems-bullmq-check.service` |
| Reverse proxy | `ems-nginx` (container) → `http://172.17.0.1:19999` |
| Bloc nginx | `nginx/conf.d/api.attendee.fr.conf`, location `/netdata/` |
| Fichier auth | `/opt/ems-attendee/backend/nginx/.htpasswd` |
| Accès host depuis container | `extra_hosts: host.docker.internal:host-gateway` (docker-compose.prod.yml) |

> Le container nginx atteint le host via l'IP du bridge Docker `172.17.0.1`. C'est volontairement une IP fixe (pas un nom DNS) pour éviter les erreurs `502` liées à la résolution DNS runtime de nginx.

---

## 5. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| `401 Unauthorized` | Mauvais identifiants | Vérifier user/mot de passe, ou réinitialiser (section 2) |
| `502 Bad Gateway` | Netdata arrêté ou injoignable | `sudo systemctl start netdata` puis vérifier `ss -tlnp \| grep 19999` |
| « Invalid Room ID » à l'invitation | Invitation tentée depuis le dashboard local | Inviter depuis `app.netdata.cloud` (section 1, Option B) |
| Nouveau mot de passe pas pris en compte | `.htpasswd` non rechargé | `sudo docker exec ems-nginx nginx -s reload` |
