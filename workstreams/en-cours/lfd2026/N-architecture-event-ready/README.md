# Chantier N — Architecture event-ready LFD 2026

> **Périmètre strict :** rendre les flux critiques solides pour LFD, avec le maximum de découplage
> utile dans le calendrier. Ce chantier coordonne I/J/B/C2.1/T/K ; il ne réécrit pas toute Attendee.

## Décision non négociable

L'inscription et le check-in restent **synchrones, courts, atomiques et autoritatifs dans PostgreSQL**.
BullMQ ne décide jamais ultérieurement si une place ou un check-in est accepté.

Après le commit seulement, BullMQ traite les effets secondaires utiles à LFD : billet/PDF, email,
notifications, statistiques non critiques, WebSocket et intégrations.

```text
requête inscription/check-in
  -> validation + transaction DB courte
  -> réponse métier immédiate
  -> jobs durables et idempotents après commit
  -> workers PDF / email / projections
```

## Livrables event

| Lot | Contenu | Dépendances | Estimation |
| --- | --- | --- | --- |
| N0 — décisions et mesure | seuil event chiffré, cartographie sync/async, baseline k6 | I/J | 0,5–1 j |
| N1 — hot paths | L9b mesuré, L9.1 check-in, L9a si flux event concerné | I | 1–2 j |
| N2 — workers | `ticket.generate`, `email.send`, rôles worker séparables, concurrence bornée | B/C2.1 | 3–5 j |
| N3 — fiabilité | idempotency keys, retry/backoff, statuts, réconciliation « job manquant », DLQ exploitable | C2.1/0-MON | 1–2 j |
| N-ANTI — anti-bot | Turnstile serveur, rate limits progressifs, idempotence de soumission, modes et tests | BIL/I/0-MON | 2,5–4,5 j |
| N4 — charge combinée | inscription + check-in + PDF/email en bruit de fond, audit O activé, saturation et backlog | J/T/O | 1–1,5 j |
| N5 — exploitation | métriques queues, pause/throttle, dégradation contrôlée, runbooks et exercice | K/0-MON | 1–1,5 j |
| **Total coordinateur** | une partie est déjà comptée dans les chantiers liés ; ne pas doubler l'effort | | **~10–17,5 j bruts** |

Détail N-ANTI : [protection anti-bot et anti-abus LFD](./N-ANTI-protection-anti-bot.md).

## Ce que je recommande

### Ordre d'exécution

1. **Comprendre et valider L9.b** : transaction, compteur `registered_count`, capacité, waitlist,
   idempotence existante et chemins de réconciliation.
2. **Lancer le k6 ciblé L9.b** : mesurer le gain réel avant d'ajouter une autre brique.
3. **Corriger le chemin réseau BIL** : éviter que le proxy PHP regroupe les visiteurs sous le quota
   unique de 100 requêtes/minute, puis poser le MVP N-ANTI.
4. **Finaliser C2.1/B** : inscription confirmée → `ticket.generate` → PDF → `email.send`, sans file
   PHP locale comme cible durable.
5. **Fermer N3** : idempotence, statuts persistants, retry/backoff, réconciliation et DLQ exploitable.
6. **Traiter L9.1** : check-in session court et atomique ; effets secondaires après commit.
7. **Fermer O avant la validation finale** : événements d'audit explicites, BIL fidèle, E2E et
   configuration de production active ; le k6 ciblé L9.b reste antérieur à O.
8. **Exécuter le k6 final combiné** : 3 000 clients actifs, inscriptions, check-ins, audit et PDF/email
   en bruit de fond ; vérifier aussi complétude et coût de l'audit.
9. **Ajouter uniquement le levier mesuré nécessaire** : cache ou portier Redis ciblé si le test échoue
   et si le goulot observé le justifie.
10. **Clore par l'exploitation** : limites de concurrence, métriques, alertes, modes dégradés et exercice
   des runbooks avant la recette.

### Recommandation de pilotage

- N reste le **seul chantier d'architecture actif pour l'événement** ;
- I/J/B/C2.1/T/K portent les implémentations, N porte leur cohérence et leurs critères de sortie ;
- ne pas compter deux fois les jours déjà estimés dans les chantiers contributeurs ;
- chaque ajout doit répondre à une exigence CDC, un incident connu ou une mesure k6 ;
- dès que les critères event sont atteints, fermer le scope et stabiliser plutôt qu'élargir.

### Frontière ferme avant LFD

Les chantiers **F et L sont entièrement hors scope avant l'événement** :

- aucun travail de statelessness générale ou de clustering issu de F ;
- aucun multi-instance API ;
- aucune HA/autoscaling GCP issue de F ;
- aucune transactional outbox généralisée ou migration globale event-driven issue de L ;
- aucun nouveau broker ;
- aucune impression/PrintNode.

Le chantier [P — plateforme de journalisation](../P-plateforme-journalisation/README.md) est lui aussi
entièrement post-event. O ne déclenche avant LFD ni l'outbox généralisée de L, ni le pipeline P.

Les idées F/L peuvent être notées lorsqu'un chantier event révèle une dette, mais elles ne deviennent
pas des tâches de N et ne consomment pas la capacité LFD. Une insuffisance k6 ne déclenche pas F/L :
elle déclenche d'abord une optimisation ciblée du goulot mesuré et un arbitrage event explicite.

## Garde-fous

- une seule instance API pour LFD tant que l'application n'est pas stateless ;
- plusieurs workers possibles seulement si la queue partagée, l'idempotence et les limites de
  concurrence sont prouvées ; commencer par un worker par rôle puis augmenter selon les métriques ;
- email/PDF ne bloquent jamais l'inscription ou le check-in ;
- un backlog secondaire ralentit le billet/email, pas l'acceptation métier ;
- aucune file locale `queue.jsonl` comme cible durable ;
- pas de portier Redis sans résultat k6 qui le justifie ; aucun multi-instance dans N ;
- pas de « vraie » plateforme event-driven/outbox généralisée avant LFD.

## Gate de décision capacité

1. Fixer la cible : volume, fenêtre d'absorption, p95/p99 et taux d'erreur acceptables.
2. Mesurer L9b/L9.1 sur staging avec le trafic combiné réaliste.
3. Si la cible est atteinte : conserver l'instance unique et fermer le scope.
4. Si elle ne l'est pas : optimiser le goulot mesuré ; évaluer Redis/isolations ciblées.
5. Le multi-instance API appartient à F et reste post-event ; il ne fait pas partie des options de N.

## Critères de sortie avant LFD

- zéro survente et zéro double check-in sous le scénario de charge retenu ;
- p95/p99 et débit conformes au seuil accepté ;
- réponse inscription/check-in indépendante de Mailgun et Gotenberg ;
- jobs secondaires idempotents, observables, rejouables et réconciliables ;
- limites de concurrence protégeant CPU, Redis et PostgreSQL ;
- test de panne d'un worker et reprise de backlog validés ;
- audit O activé dans la configuration testée, sans perte/doublon sur les événements attendus ;
- runbooks `pause`, `resume`, `drain`, `retry`, `DLQ`, dégradation et rollback utilisables le jour J.

## Ce qui est explicitement reporté

- impression et PrintNode : hors scope LFD, à traiter après l'événement dans F/L ;
- API stateless et clustering : chantier F ;
- transactional outbox généralisée, contrats et événements métier : chantier L ;
- plateforme de journalisation, sinks et centralisation technique : chantier P ;
- HA/autoscaling GCP : chantier F/migration GCP.
