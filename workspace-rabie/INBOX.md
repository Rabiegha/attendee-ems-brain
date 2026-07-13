# INBOX — Capture rapide

> 1 ligne, taggée, sans réfléchir. On trie en fin de session.
> Tags : `[bug]` `[idea]` `[trap]` `[learn]` `[archi]`

## À trier

- 🔴 [ops][decision] **Branch protection main/staging bloquée : GitHub Free ne la permet pas sur repos privés** (constat 13/07, API + rulesets = 403). Décider : **GitHub Pro ~4 $/mois compte Rabiegha** (reco) vs org Team vs rien. Dès upgrade → me redemander, les commandes `gh api` sont prêtes. Détail → [finalisation §5](../workstreams/en-cours/lfd2026/0-fondations/finalisation-ci-cd-et-livraison.md).
- 🧹 [ops] Branches supprimables après vérif : `chore/ci-cd` (back+front, tout mergé le 13/07) + `infra/staging-stack` (mergée). staging = main partout.
- ⏸ [ops] **CD prod back + CD prod front : EN ATTENTE** (validés côté staging le 13/07) — fenêtre 22h-06h ou `hotfix=true`. ⚠️ `main` contient désormais tout `staging` (merge 13/07) : un deploy prod déploiera le nouveau code, pas un no-op. → reprendre depuis [finalisation-ci-cd-et-livraison.md](../workstreams/en-cours/lfd2026/0-fondations/finalisation-ci-cd-et-livraison.md) §3.
- 🔴 [ops] **MERCREDI 15/07 (très important) : purchase du plan Mailgun (32 €/mois)** — bloqué le 13/07 par un bug billing Mailgun (404 sur `/v1/mailforce/*`, bouton Purchase désactivé). Pas bloquant : le warm-up démarre en compte starter en attendant. Si toujours KO mercredi → support Mailgun. Détail → journal [2026-W29](./journal/2026-W29.md).
- 🚨 [bug][security] **Webshells PHP injectés dans `choyou.fr/_/attendee/`** (mutualisé legacy, PAS le VPS). Vérifier l'arborescence + supprimer les fichiers non désirés ; ils réapparaissent → persistance probable (changer creds FTP/SSH/DB, vérifier crons OVH). 2 fichiers déjà supprimés le 03/07. → `bugs/a-faire/2026-07-03-webshell-php-choyou-attendee.md`
- [archi] back/LFD : **plafond inscription ≈ 33–37 reg/s = sérialisation écriture DB** (prouvé : 4 cœurs = même débit que 2). Levier = sortir le COUNT capacité de la transaction. → workstream infra-scaling-pca
- [trap] back/LFD : cluster N instances **casse l'impression temps réel silencieusement** (registres en mémoire) → JAMAIS en prod sans clustering (redis-adapter+sticky+présence Redis).
- [learn] back/LFD : email post-inscription = **QR code** dans le mail, **pas** un badge PDF (le PDF est généré à l'impression). Corriger `lfd-2026-executive-summary.md`.
- [trap] back : le VPS déploie par **copie de fichiers (scp) + rebuild**, pas `git pull`. VPS tourne sur `main`, dev sur `staging`.
- [idea] back/LFD : refaire une mesure k6 **propre** depuis une machine externe (les chiffres actuels sont un plancher, k6 co-localisé avec l'API).
- [bug] mobile : éjection vers login au hard restart de l'app → workstream mobile-stabilization/01
- [bug] mobile : éjection quand on ouvre l'écran de scan → workstream mobile-stabilization/02
- [archi] vérifier si un PersistGate entoure bien la navigation avant lecture de l'auth state
- [learn] appVersionSource = remote → ne pas se fier au versionCode local (cf mémoire repo)
- [idea] context : implémenter `--annotate` sur get-context.js (fraîcheur inline dans le bundle IA) → workstream context-system/01
- [idea] context : écrire la doc 1 page hub/README.md (pitch « IA qui connaît le projet ») → workstream context-system/03

## Trié (archive courte)

<!-- déplacer ici une ligne quand elle a été rangée, puis nettoyer régulièrement -->
