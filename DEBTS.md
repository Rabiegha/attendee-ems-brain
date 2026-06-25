# DEBTS — Dettes techniques et sécurité

Dettes connues à ne pas oublier. Ne **pas** corriger sans ticket explicite.

> Ce fichier est la version **lisible par toi** des notes internes de l'agent.
> Règle : tout ce que Copilot note en mémoire privée qui te concerne doit aussi atterrir ici.

---

## Sécurité

| # | Dette | App | Sévérité | À faire |
|---|---|---|---|---|
| S-01 | `EXPO_PUBLIC_PRINTNODE_API_KEY` en clair dans `eas.json` (4 profils), `.env` et `.env.local-backup` — commité sur GitHub. | mobile | ⚠️ Haute | 1. Générer nouvelle clé PrintNode. 2. `eas env:create` → EAS Secrets. 3. Retirer de `eas.json` et `*.env*`. |
| S-02 | Redis prod (`ems-redis`) sans mot de passe malgré `$REDIS_PASSWORD` dans l'env. | back/infra | ⚠️ Haute | Configurer l'auth Redis dans le container + mettre à jour le docker-compose prod. |
| S-03 | IP réelle du client **non transmise** au backend (Nginx ne forward pas `X-Forwarded-For`). `refresh_tokens.ip` enregistre toujours l'IP du container Nginx `172.18.0.7`. | back/infra | 🟡 Moyenne | Configurer `trust proxy` dans Nest + `proxy_set_header X-Forwarded-For` dans Nginx. |

---

## Technique — Back / Scaling (LFD 2026)

| # | Dette | App | Sévérité | À faire |
|---|---|---|---|---|
| T-10 | **Plafond inscription ≈ 33–37 reg/s** = sérialisation sur la transaction d'écriture (`registerToEvent`), notamment le `registration.count(...)` de capacité in-transaction. Prouvé : 4 cœurs = même débit que 2 (367 % CPU). | back | 🟠 Haute | Sortir le COUNT du chemin chaud (compteur dénormalisé / verrou ciblé) + raccourcir la transaction au strict `create`. Préserver l'anti-surbooking. Détail : `attendee-ems-back/docs/infra/lfd-2026-load-test-results.md` §5. |
| T-11 | **Cluster API (Voie B) interdit en prod** : multi-instances casse l'impression temps réel **silencieusement** (registres présence/imprimantes en mémoire de process). | back/infra | 🔴 Critique | Pré-requis avant tout cluster prod : `@socket.io/redis-adapter` + sticky sessions + présence externalisée Redis. Voir workstream `api-scaling-clustering`. NE PAS ajouter de replicas à `docker-compose.prod.yml`. |
| T-12 | Doc `lfd-2026-executive-summary.md` dit « badge **PDF** dans l'email » — faux, c'est un **QR code** (le PDF est généré à l'impression). | back/docs | 🟡 Basse | Corriger avant transmission client. |
| T-13 | Correctif **pgBouncer prod** (`LISTEN_PORT 6432`) non déployé. Containers `ems-api`/`ems-staging-api` affichés `unhealthy` mais renvoient `200` (faux-positif healthcheck). | back/infra | 🟡 Moyenne | Déployer le correctif pgBouncer prod ; ajuster le healthcheck. |

---

## Technique — Back / Authz

| # | Dette | App | Sévérité | À faire |
|---|---|---|---|---|
| T-02 | **23 permissions** avec `scope='org'` encore en base prod. Doublons (équivalent `:any` existe). Workarounds en place : `scope-evaluator.ts` L20 + `permission-mapper.ts`. Prod fonctionne mais c'est fragile. | back + front | 🟠 Haute | Plan B complet : dump SQL avant → DELETE doublons → UPDATE vers `any` → DELETE `scope='org'` → PR back (retirer `case 'org'`) → PR front (idem) → test CI anti-drift. Voir mémoire `authz-v1-debt.md` pour le SQL préparé. |

---

## Technique — Print Client

| # | Dette | App | Sévérité | À faire |
|---|---|---|---|---|
| T-03 | **D-V2-03 🟠 CRITIQUE** : faux positif COMPLETED si imprimante offline. `webContents.print` retourne success dès que CUPS accepte le spool, même si l'imprimante est déconnectée. Repro job `09614e38`. | print-client | 🟠 Haute | Mitigation Phase 2/3 : `lpstat` sanity-check avant PATCH `/complete`. |
| T-04 | **UX-1 (mobile)** : double notification contradictoire — toast "badge imprimé" optimiste + WS FAILED "cancelled" arrivent tous les deux. | mobile | 🟡 Moyenne | Sur réception `print-job:updated` FAILED, écraser/supprimer le toast succès précédent (clé = jobId ou registrationId). |
| T-05 | **UX-2 (print-client)** : job FAILED via cancel API n'apparaît pas dans l'historique UI à la reconnexion. | print-client | 🟡 Moyenne | Vérifier si `GET /print-queue/history` inclut les FAILED. Si oui → bug renderer (filtre status). Sinon → ajouter FAILED à l'endpoint. |
| T-06 | **UX-3 (print-client)** : dialog OFFLINE jobs propose seulement "Imprimer tout" / "Annuler tout". Pas de sélection granulaire. | print-client | 🟡 Basse | Phase 4+ : checkboxes par job dans le dialog. |
| T-07 | D-016 : pas de statut CANCELLED dédié côté client. | print-client | 🟡 Basse | Phase 4+. |
| T-08 | D-017 : pas de kill-switch WS PRINTING. | print-client | 🟡 Basse | Phase 4+. |
| T-09 | D-V2-02 : timeout bannière mobile 10s → devrait être 30s + polling fallback. | mobile | 🟡 Basse | Workstream mobile stabilization. |

---

## Technique — Infra / Deploy

| # | Dette | App | Sévérité | À faire |
|---|---|---|---|---|
| T-10 | `sudo bash deploy-back.sh` **échoue** en prod : sudo perd la SSH config de `debian` → `Permission denied (publickey)`. | infra | 🟡 Moyenne | Adapter le script pour exécuter le `git pull` en tant que `debian` (pas sudo) puis relancer Docker en sudo. |
| T-11 | `eas-cli` local v14.x, dernier = v20. Pas bloquant. | mobile | 🟢 Basse | `npm install -g eas-cli@latest` quand pratique. |

---

## Règle de mise à jour

- Quand Copilot dit "c'est dans ma mémoire" → demander à le coller ici aussi.
- Quand une dette est résolue → la supprimer de ce fichier + noter dans [learnings/](learnings/README.md) si pertinent.
