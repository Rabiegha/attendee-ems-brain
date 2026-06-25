# INBOX — Capture rapide

> 1 ligne, taggée, sans réfléchir. On trie en fin de session.
> Tags : `[bug]` `[idea]` `[trap]` `[learn]` `[archi]`

## À trier

- [archi] back/LFD : **plafond inscription ≈ 33–37 reg/s = sérialisation écriture DB** (prouvé : 4 cœurs = même débit que 2). Levier = sortir le COUNT capacité de la transaction. → workstream api-scaling-lfd2026
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
