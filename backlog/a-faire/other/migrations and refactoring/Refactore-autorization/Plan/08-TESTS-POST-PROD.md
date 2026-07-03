# Tests post-production

> **Quand :** immédiatement après chaque phase de deploy + à J+1, J+3, J+7.
> **Qui :** la personne qui a déployé + un binôme.

## Objectif

Confirmer que la prod fonctionne **réellement**, pas juste que ça boote. Détecter les régressions silencieuses qui ne plantent pas mais dégradent l'expérience.

---

## A. Smoke tests immédiats (T+5 min après deploy)

### A.1 — Health & infra

```bash
# Back
curl -fsS https://api.choyou.fr/health || echo "DOWN"
# Front
curl -fsS -o /dev/null -w "%{http_code}\n" https://app.choyou.fr/  # attendu 200
```

### A.2 — Authentification (compte de test prod dédié)

⚠️ **Avoir un compte de test dédié en prod**, pas un compte client.

```bash
# Login
TOKEN=$(curl -s -X POST https://api.choyou.fr/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"qa-test@choyou.fr","password":"***"}' | jq -r .access_token)
[ -z "$TOKEN" ] && echo "LOGIN KO" || echo "LOGIN OK"

# me/orgs
curl -fsS -H "Authorization: Bearer $TOKEN" https://api.choyou.fr/auth/me/orgs | jq

# /me/ability
curl -fsS -H "Authorization: Bearer $TOKEN" https://api.choyou.fr/me/ability | jq
```

### A.3 — UI manuelle (web)

- [ ] Ouvrir https://app.choyou.fr → page de login OK
- [ ] Login compte test → dashboard OK
- [ ] Switch d'organisation → OK
- [ ] Logout → retour login OK

### A.4 — UI manuelle (mobile)

- [ ] Ouvrir l'app sur iOS → login OK
- [ ] Idem Android
- [ ] Session restaurée après kill app

---

## B. Tests fonctionnels (T+30 min)

Plus complets, sur les flux principaux.

### B.1 — Sur compte root

- [ ] Liste des orgs (3+ visibles)
- [ ] Switch sur une org sans membership → bypass actif, accès aux pages
- [ ] Création d'un user dans une org → OK
- [ ] Lecture des logs admin → OK

### B.2 — Sur compte admin org

- [ ] Voit uniquement son org
- [ ] Peut créer un event
- [ ] Peut inviter un user
- [ ] Tente d'accéder à une autre org → 403 ou redirection

### B.3 — Sur compte agent

- [ ] Voit uniquement les events qui lui sont assignés
- [ ] Peut scanner les attendees de ses events
- [ ] Tente de créer un event → 403

---

## C. Surveillance continue (T+1h, T+24h, T+72h, T+7j)

### C.1 — Dashboards

- [ ] **Sentry back** : taux d'erreurs vs baseline (avant deploy)
  - Critère : pas de spike > 2x la baseline
  - Erreurs nouvelles ? Investiguer immédiatement
- [ ] **Sentry front** : idem
- [ ] **Sentry mobile** : crash-free rate > 99%
- [ ] **Cloud Run logs** :
  - Pas de boucle d'erreur 500
  - Latence p50/p95/p99 stable
- [ ] **Cloud SQL** :
  - CPU stable
  - Connexions stables
  - Slow queries pas en hausse

### C.2 — Métriques business

- [ ] Nombre de logins / heure : conforme aux moyennes habituelles
- [ ] Taux d'échec login : pas de hausse
- [ ] Nombre d'erreurs 401 : stable
- [ ] Nombre d'erreurs 403 : pas de hausse anormale (signe que le RBAC casse)
- [ ] Temps de session moyen : stable

---

## D. Tests de régression (T+24h)

Refaire la matrice de `04-TESTS-E2E-INTEGRATION.md` mais en prod sur le compte de test, pas en local. Cocher chaque ligne.

---

## E. Tests utilisateur (T+72h)

- [ ] Demander à 3-5 utilisateurs internes de tester leur workflow habituel
- [ ] Récolter feedback bugs / lenteurs
- [ ] Surveiller le support : nouveaux tickets liés à login/permissions ?

---

## F. Validation finale (T+7j)

Si tous les voyants sont au vert pendant 7 jours :

- [ ] Marquer le refactor comme stabilisé
- [ ] Lancer la préparation du **Palier 4** (`05-PALIER-4-cleanup-http-deprecated.md`)
- [ ] Archiver les backups manuels (garder le dump pré-refactor au moins 30j)
- [ ] Mettre à jour la documentation interne

---

## Critères d'alerte (à n'importe quel moment)

🚨 **Rollback immédiat si :**

- Taux d'erreur back > 5% pendant > 5 min
- Login impossible pour > 10% des utilisateurs
- Crash mobile > 1% sur une nouvelle version
- Erreur Prisma sur `user_roles` ou `user_system_access`
- 5xx en cascade sur les endpoints `/auth/*`

🟡 **Investigation urgente si :**

- Spike Sentry x2 vs baseline
- Latence p95 > 2x normale
- Nouveau type d'erreur jamais vu avant deploy

🟢 **Surveiller normalement si :**

- Quelques erreurs isolées (à investiguer mais pas urgent)
- Métriques dans la marge habituelle

---

## Modèle de rapport post-deploy

À remplir et archiver dans `docs/Refactore-autorization/post-mortems/` :

```markdown
# Post-deploy report — Refactor Authz v1 — Phase X

**Date deploy :** YYYY-MM-DD HH:MM
**Durée maintenance :** X minutes
**Personne qui a déployé :** XXX
**Binôme :** XXX

## Résultat
- [ ] OK
- [ ] OK avec incidents mineurs (cf section Incidents)
- [ ] Rollback (cf section Rollback)

## Métriques avant/après
| Métrique | Avant | Après deploy | T+24h |
|---|---|---|---|
| Taux erreur back | X% | X% | X% |
| Latence p95 | Xms | Xms | Xms |
| Login/heure | X | X | X |

## Incidents
(Aucun / liste avec horodatage)

## Décision
- [ ] Continuer Phase suivante
- [ ] Pause + investigation
- [ ] Rollback
```
