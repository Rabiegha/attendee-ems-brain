# 08 — Things to Avoid

> **STATUS : TO REVIEW / DISCOVERY**

Liste exhaustive des anti-patterns à ne **jamais** introduire dans la solution cible. Chaque entrée est traçable dans [10-acceptance-criteria-draft.md](./10-acceptance-criteria-draft.md).

---

## A. Côté donnée / modèle

1. **Ne pas mettre `attendee.id` directement dans le QR code.**
   Un attendee peut être recréé, fusionné, ou son ID régénéré.

2. **Ne pas dépendre uniquement de `registration.id` pour le scan.**
   C'est précisément le problème actuel. La résolution doit pouvoir passer par `stable_identity_key`.

3. **Ne pas exposer un identifiant technique (UUID) dans une URL publique.**
   Le QR public d'aujourd'hui (`/qrcode/:registrationId`) doit être déprécié.

4. **Ne pas considérer `email` seul comme une clé immuable.**
   L'email peut changer. Si c'est la seule clé disponible, fallback documenté et `needs_review` plutôt que casse silencieuse.

5. **Ne pas stocker la logique de validité QR dans le frontend.**
   Le mobile / front ne décide pas. Le backend est l'unique source de vérité.

6. **Ne pas utiliser Redis comme source de vérité pour les QR.**
   Redis peut être cache, jamais persistance principale. La DB reste autoritative.

---

## B. Côté cycle de vie

7. **Ne pas hard-deleter une registration qui a un QR émis, envoyé ou imprimé.**
   Soft cancel obligatoire, avec révocation du QR si besoin.

8. **Ne pas considérer un ancien QR connu comme invalide automatiquement.**
   S'il est résolvable, il doit être accepté (use case 7).

9. **Ne pas régénérer les QR à chaque update.**
   Une modification de nom/entreprise ne doit pas invalider un QR déjà envoyé.

10. **Ne pas régénérer les QR à chaque réimport.**
    La réconciliation d'identité doit retrouver la registration et préserver le credential.

11. **Ne pas créer plusieurs QR actifs concurrents pour la même registration sans règle explicite.**
    Si multi-QR un jour, modèle de priorité ou d'aliasing obligatoire.

12. **Ne pas supprimer l'historique des QR.**
    Les QR `revoked`, `replaced`, `expired` restent en base pour audit et résolution.

---

## C. Côté import

13. **Ne pas faire `delete-all + recreate` comme stratégie d'import par défaut.**
    L'import doit être un **upsert** avec réconciliation d'identité.

14. **Ne pas ignorer les doublons d'import.**
    Doublon = `needs_review`, pas écriture silencieuse de deux registrations.

15. **Ne pas accepter d'import sans aucune clé d'identification fiable.**
    Si seulement nom + entreprise, refuser ou avertir explicitement.

16. **Ne pas absorber silencieusement un import qui produit des `needs_review`.**
    Le rapport doit être visible côté UI organisateur.

---

## D. Côté scan / UX

17. **Ne pas retourner « QR replaced » sans permettre d'action terrain.**
    Toute réponse de scan doit être exploitable par l'hôtesse.

18. **Ne pas afficher de message technique à l'hôtesse.**
    Pas de `UNRESOLVED_TOKEN_E42`, pas d'UUID, pas de stack trace.

19. **Ne pas casser silencieusement.**
    Un QR qui ne résout pas doit générer un message clair + une piste d'action (recherche par nom).

20. **Ne pas faire confiance uniquement au payload du QR.**
    Validation systématique côté backend, même si le payload est signé.

21. **Ne pas mettre le mobile en autonomie de décision.**
    Le mobile affiche, n'arbitre pas.

---

## E. Côté sécurité

22. **Ne pas exposer le `token` du QR dans des logs accessibles.**
    Logs structurés avec masquage si nécessaire.

23. **Ne pas réutiliser un `token` après révocation.**
    Un token révoqué reste révoqué à vie.

24. **Ne pas implémenter un bouton « forcer le check-in » sans audit, rôle et raison obligatoire.**

25. **Ne pas faire de QR signé (JWT) sans plan clair de rotation et de révocation.**
    Sinon piège de la stateless authentication.

---

## F. Côté architecture

26. **Ne pas traiter ce sujet comme un simple problème UI.**
    C'est un problème de **modèle d'identité** et de **cycle de vie**.

27. **Ne pas concentrer toute la logique dans un seul service monolithique.**
    Le scan, l'émission, la réconciliation d'import sont trois responsabilités distinctes.

28. **Ne pas ignorer la compatibilité future event-driven.**
    L'émission d'un QR doit pouvoir être déclenchée par un événement domaine.

29. **Ne pas casser la compatibilité avec les workers BullMQ futurs.**
    L'émission async (email + QR) doit être idempotente.

30. **Ne pas livrer en big-bang.**
    Phasage obligatoire (cf. [11-future-implementation-plan-draft.md](./11-future-implementation-plan-draft.md)).
