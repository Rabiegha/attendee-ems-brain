# 09 — Open Questions

> **STATUS : TO REVIEW / DISCOVERY**

Questions à trancher avant tout démarrage d'implémentation. Les réponses guideront le scope précis des Phases 0 à 7 ([11-future-implementation-plan-draft.md](./11-future-implementation-plan-draft.md)).

## Légende

- **Category** : Audit (Q sur l'existant) · Product (Q produit/business) · Tech (Q technique/architecture) · UX · Migration · Security.
- **Status** : `OPEN`, `IN_DISCUSSION`, `DECIDED`, `DEFERRED`.

---

| # | Question | Category | Why it matters | Options | Recommended default | Status |
|---|---|---|---|---|---|---|
| Q1 | Quelle est exactement la donnée encodée dans les QR aujourd'hui ? | Audit | Permet de mesurer la surface de migration. | (a) `registration.id` brut · (b) `external_id` · (c) URL signée · (d) autre | (a) confirmé pour email/badge, à valider pour autres surfaces | OPEN |
| Q2 | Les QR existants sont-ils déjà envoyés à des vrais utilisateurs ? | Product | Détermine la nécessité d'une rétro-compatibilité prolongée. | (a) Oui en prod, plusieurs clients · (b) Oui mais peu de volume · (c) Non, encore en pilote | (a) à supposer en prod | OPEN |
| Q3 | Le système utilise-t-il `attendee.id`, `registration.id` ou autre ? | Audit | Définit la cible exacte à déprécier. | (a) `registration.id` · (b) `attendee.id` · (c) mix | (a) confirmé | DECIDED |
| Q4 | Existe-t-il déjà un champ stable côté attendee/registration ? | Audit | Détermine si on promeut l'existant ou on en crée un nouveau. | (a) `external_id` existant · (b) aucun · (c) plusieurs candidats | (a) `external_id` existe, sémantique floue | OPEN |
| Q5 | Quelle clé de matching est la plus fiable chez les clients ? | Product | Influence la priorité des sources de `stable_identity_key`. | (a) `external_reference` CRM · (b) `employee_id` · (c) email · (d) autre | (a) si fournie, (c) sinon | OPEN |
| Q6 | Les imports ont-ils une colonne `external_reference` explicite ? | Product | Si oui, on peut s'appuyer dessus immédiatement. | (a) Toujours · (b) Souvent · (c) Rarement · (d) Variable selon client | (d) probablement | OPEN |
| Q7 | Que faire si l'email change entre deux imports ? | Product | Cas fréquent en pratique. | (a) Conserver via external_ref · (b) `needs_review` · (c) Créer nouvelle registration · (d) Bloquer l'import | (a) + (b) en fallback | OPEN |
| Q8 | Que faire si deux lignes ont le même email dans un import ? | Product | Question récurrente côté support. | (a) Refuser l'import · (b) Garder la première · (c) `needs_review` sur les deux · (d) Demander à l'organisateur | (c) | OPEN |
| Q9 | Le hard delete d'une registration doit-il être interdit après QR envoyé ? | Product | Cause racine du problème. | (a) Interdit · (b) Interdit sauf override admin · (c) Autorisé avec warning · (d) Autorisé comme aujourd'hui | (b) | OPEN |
| Q10 | Un ancien QR doit-il toujours être accepté si la registration existe ? | Product | Définit la sémantique d'`alias_resolved`. | (a) Oui systématiquement · (b) Oui sauf si `revoked`/`blocked` · (c) Oui avec confirmation · (d) Non | (b) | OPEN |
| Q11 | Quand faut-il bloquer un QR ? | Product | Liste des cas de `blocked`. | Cas à lister | À définir avec produit | OPEN |
| Q12 | Faut-il un mode `supervisor_review` distinct de `blocked` ? | UX | Sépare blocage dur de demande de validation humaine. | (a) Oui · (b) Non | (a) | OPEN |
| Q13 | Faut-il historiser les QR envoyés (`qr_sent_at`, channel, template) ? | Product | Nécessaire pour les règles de protection contre suppression. | (a) Oui complet · (b) Oui basique · (c) Non | (a) | OPEN |
| Q14 | Faut-il supporter plusieurs QR actifs par registration ? | Product | Cas badge multiples / multi-jours. | (a) Oui dès V1 · (b) Plus tard · (c) Jamais | (b) | OPEN |
| Q15 | Comment gérer les anciens QR déjà générés (rétro-compat) ? | Migration | Évite de casser la prod. | (a) Backfill `QrCredential` à partir de `registration.id` · (b) Période de grace au scan · (c) Combinaison · (d) Rien | (c) | OPEN |
| Q16 | Le mobile doit-il supporter le scan offline ? | Product | Influence le choix Option E (JWT). | (a) Oui V1 · (b) V2+ · (c) Non | (b) | DEFERRED |
| Q17 | Quelle UX pour l'hôtesse en cas d'ambiguïté (`supervisor_review`) ? | UX | Critique jour J. | (a) Modal bloquante · (b) Bouton « Appeler superviseur » · (c) File d'attente backoffice | (b) + (c) | OPEN |
| Q18 | Faut-il dépréquer la route publique `/qrcode/:registrationId` ? | Security | Aujourd'hui non authentifiée, expose l'UUID. | (a) Déprécier · (b) Garder + signer · (c) Garder | (a) | OPEN |
| Q19 | Quel format de token pour `QrCredential.token` ? | Tech | Lisibilité, entropie, longueur QR. | (a) UUIDv4 · (b) NanoID 21 chars · (c) Base32 24 chars · (d) JWT | (b) ou (c), entropie ≥ 128 bits | OPEN |
| Q20 | Doit-on stocker la source effective de la `stable_identity_key` ? | Tech | Utile pour debug et audit. | (a) Oui (`stable_identity_source` enum) · (b) Non | (a) | OPEN |
| Q21 | Comment normaliser les emails pour le matching ? | Tech | Évite les faux négatifs (case, accents, alias Gmail). | (a) `lower(trim(email))` · (b) + dotless local part Gmail · (c) + Punycode IDN | (a) en V1, (b)/(c) plus tard | OPEN |
| Q22 | Comment gérer un téléphone comme clé d'identité ? | Tech | Normalisation E.164 nécessaire. | (a) E.164 strict · (b) Best effort | (a) | OPEN |
| Q23 | Faut-il un endpoint admin pour révoquer un QR ? | Product | Use case réel (compromission). | (a) Oui · (b) Non, via support | (a) | OPEN |
| Q24 | Faut-il logger tous les scans (audit) ? | Security | Permet l'analyse post-mortem. | (a) Oui systématique · (b) Échantillon · (c) Erreurs uniquement | (a) | OPEN |
| Q25 | RGPD : durée de rétention des `QrCredential` `revoked` ? | Security | Définit la politique de purge. | (a) Durée de l'événement + N mois · (b) X années · (c) Indéfini | (a) | OPEN |
| Q26 | Compatibilité event-driven : le scan doit-il publier un événement domaine ? | Tech | Aligne avec le workstream async. | (a) Oui dès V1 · (b) Plus tard · (c) Non | (a) lecture seule (event log) | OPEN |
| Q27 | Backfill : qui décide la `stable_identity_key` pour les registrations existantes ? | Migration | Risque de fausses fusions. | (a) `external_id` si présent sinon `normalized_email` · (b) Manuel client · (c) Script + revue | (a) avec rapport | OPEN |
| Q28 | Faut-il une UI organisateur pour visualiser les QR émis ? | Product | Améliore la confiance et le support. | (a) Oui · (b) Non V1 | (b) | DEFERRED |
| Q29 | Faut-il bloquer le réimport si > N % de `needs_review` ? | Product | Sécurise contre les fichiers cassés. | (a) Oui seuil produit · (b) Avertir uniquement | (b) | OPEN |
| Q30 | Multi-tenant : un QR peut-il appartenir à plusieurs orgs ? | Tech | Important pour le scoping. | (a) Non, jamais · (b) Oui si event partagé | (a) | OPEN |

## Suivi

Chaque question doit être tranchée avant d'entrer en Phase 1 d'implémentation. Les questions DEFERRED sont à re-poser au début de chaque release suivante.
