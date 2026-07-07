# Learnings

Ce que j'apprends, pour ne pas refaire les mêmes erreurs. Vient de l'inbox `[learn]`.

> **Pas de découpe fait / en cours / à faire ici** : un apprentissage est **acquis**, ce n'est pas une tâche.

| Date | Apprentissage |
|---|---|
| 2026-06-09 | `appVersionSource: remote` → ignorer le `versionCode` local d'`app.json` (cf mémoire repo mobile). |
| 2026-07-06 | [Séparer reformatage et logique](2026-07-06-reformatage-vs-logique.md) : garder des diffs chirurgicaux (1 levier = 1 commit) en évitant le reformatage auto (format-on-save, `npm run format`). |
| 2026-07-07 | [Stack staging — concepts infra](2026-07-07-stack-staging-concepts-infra.md) : vhost, pgBouncer vs pool/`connection_limit`, `directUrl` Prisma, pattern `.env` template/secrets, piège password ↔ volume Postgres, pourquoi rejouer L1/L2/L10 ne casse rien. |
| 2026-07-07 | [pgBouncer (L2) vs pool/`connection_limit` (L10)](2026-07-07-pgbouncer-et-pool-db.md) : deux couches de connexions DB, c'est quoi un « client », où se règle chaque limite. |
| 2026-07-07 | [`directUrl` Prisma (L3)](2026-07-07-directurl-prisma.md) : deux URLs (runtime via pgBouncer / migrations en direct), pourquoi le mode transaction casse les migrations. |
| 2026-07-07 | [Git : `checkout <ref> -- fichier` & `reset --soft`](2026-07-07-git-checkout-ref-fichier.md) : checkout écrase (ne fusionne pas) vs cherry-pick/merge ; reset --soft rembobine en gardant le travail ; piège `--amend` après push → doublon → réparation `reset --soft` + `--force-with-lease`. |
