# Suivi operationnel — Gotenberg Cloud Run + Auth

> Chantier B0/B1 — LFD 2026  
> Derniere mise a jour : 2026-07-16  
> Objectif du creneau : terminer la mise en place Cloud Run prive + auth OIDC, puis valider par smoke test.

---

## Etat express

| Sujet | Statut | Note |
| --- | --- | --- |
| POC local Gotenberg | Fait | PDF genere, rendu global OK, ecart surtout anti-aliasing texte. |
| Architecture cible | Fait | Nest/OVH appelle Cloud Run en HTTPS, Cloud Run retourne les bytes PDF, pas de rappel vers OVH. |
| Cloud Run Gotenberg | Bloque cote GCP jusqu'au retour equipe | Service a deployer/configurer avec acces prive. Call reporte a mardi prochain. |
| Auth Cloud Run | Bloque pour validation reelle | Cloud Run doit refuser l'anonyme et accepter uniquement un token OIDC valide. |
| Integration Nest/OVH | A preparer sans attendre | Preparer le code/env pour generer le token OIDC et appeler Gotenberg avec `Authorization: Bearer <token>`. |
| Smoke test | A planifier mardi | Valider refus sans token + PDF OK avec token. Objectif : arriver mardi avec tout pret cote Rabie. |

---

## Suite apres report du call GCP

Le call etant reporte a mardi prochain, l'objectif cote Rabie n'est pas d'attendre : il faut
preparer tout ce qui ne depend pas de l'URL Cloud Run reelle, pour que mardi soit uniquement un
test d'assemblage.

### Ce que je peux faire sans URL Cloud Run

| Action cote Rabie | Faisable sans URL ? | Pourquoi / livrable |
| --- | --- | --- |
| Definir les variables d'environnement | Oui | Preparer `GOTENBERG_URL`, `GOTENBERG_AUDIENCE`, `GOTENBERG_AUTH_MODE`, `GOTENBERG_TIMEOUT_MS`, `GOTENBERG_ENABLED`. Les valeurs reelles seront injectees mardi. |
| Preparer le client HTTP Gotenberg | Oui | Code qui poste le HTML vers `/forms/chromium/convert/html`, avec timeout, erreurs lisibles, logs, et retour `Buffer`. |
| Preparer l'interface de rendu PDF | Oui | Pouvoir choisir `puppeteer` ou `gotenberg` via config, sans casser le fallback local. |
| Garder Puppeteer en fallback | Oui | Si Cloud Run echoue ou est desactive, le systeme continue avec le chemin actuel. |
| Preparer la generation du token OIDC | Partiellement | On peut coder le provider et les tests mockes. Le token final depend de l'audience reelle Cloud Run. |
| Preparer les tests unitaires/mock | Oui | Simuler `200 PDF`, `401/403`, timeout, 5xx, fallback Puppeteer. |
| Preparer le script de smoke test | Oui | Script parametrable par env : des que l'URL et le token sont disponibles, on lance. |
| Preparer le HTML minimal de test | Oui | Fichier `index.html` simple pour valider Gotenberg independamment du template final. |
| Mesurer le payload template reel | Oui | Exporter/generer un HTML de badge, mesurer HTML + assets + fonts pour anticiper les limites Cloud Run. |
| Rejouer le comparateur local | Oui | Continuer la validation rendu Puppeteer vs Gotenberg local sur template final. |

### Ce qui reste bloque jusqu'a reception des infos GCP

| Point | Bloque par quoi ? | Pourquoi |
| --- | --- | --- |
| Generer un token OIDC valide final | URL/audience Cloud Run exacte | L'audience du token doit matcher le service Cloud Run. |
| Tester `--no-allow-unauthenticated` | Service Cloud Run deploye | Il faut l'URL reelle pour verifier le refus anonyme `401/403`. |
| Tester `roles/run.invoker` | IAM configure cote GCP | Sans role, le token peut etre genere mais l'appel sera refuse. |
| Valider le PDF via Cloud Run | Service deploye + URL | Le rendu local ne prouve pas l'appel Cloud Run prive. |
| Decider allowlist IP OVH | Reponse equipe GCP | C'est une policy infra/securite cote GCP. |
| Mettre les env staging/prod definitives | URL + audience + credential final | On peut preparer les noms, pas les valeurs finales. |

### Ordre conseille avant mardi

1. Preparer le client Gotenberg cote Nest avec config env et fallback Puppeteer.
2. Preparer la couche auth OIDC derriere une interface, meme si l'audience reelle manque.
3. Ajouter des tests mockes : success PDF, 401/403, timeout, fallback.
4. Preparer un script de smoke test parametrable par `GOTENBERG_BASE_URL`.
5. Demander a l'equipe GCP d'envoyer les infos des que le service est pret, idealement avant mardi.
6. Mardi : injecter URL/audience/credential, lancer smoke test ensemble, puis brancher staging.

### Infos minimales a demander a l'equipe GCP

- URL Cloud Run du service Gotenberg.
- Region et projet GCP.
- Nom/email du service account appelant.
- Confirmation que `--no-allow-unauthenticated` est actif.
- Confirmation que le service account appelant a `roles/run.invoker`.
- Decision sur l'allowlist IP OVH : necessaire ou non en plus de l'OIDC.
- Mode conseille pour le credential cote OVH/Nest : cle service account temporaire, impersonation, ou autre mecanisme prefere par leur policy.

### Brouillon de reponse mail

Objet : Cloud Run Gotenberg - preparation avant le call de mardi

Bonjour,

Merci pour l'information, bien note pour le report du call a mardi prochain.

De mon cote, pour eviter d'attendre mardi et avancer en parallele, je vais preparer l'integration
cote OVH/Nest avec des variables d'environnement et un smoke test parametrable.

Pour que nous puissions faire un test complet ensemble mardi, pouvez-vous me transmettre des que
possible, idealement avant le call :

- l'URL Cloud Run du service Gotenberg ;
- la region et le projet GCP utilises ;
- le nom/email du service account appelant ;
- la confirmation que le service est bien configure en `--no-allow-unauthenticated` ;
- la confirmation que le service account appelant a le role `roles/run.invoker` ;
- votre decision sur l'allowlist IP OVH : necessaire ou non en plus de l'OIDC ;
- le mode recommande pour le credential cote OVH/Nest afin de generer le token OIDC.

Des que j'ai l'URL Cloud Run, je pourrai la renseigner cote Nest comme audience OIDC et preparer le
smoke test. L'objectif serait que mardi nous ayons uniquement a valider ensemble :

1. appel sans token refuse (`401/403`) ;
2. appel avec token OIDC accepte ;
3. generation PDF OK via Gotenberg ;
4. logs Cloud Run visibles.

Merci,
Rabie

---

## Priorite des prochaines 15 minutes

### T+0 a T+5 — verrouiller Cloud Run prive

- [ ] Eux / GCP : confirmer l'URL Cloud Run du service Gotenberg.
- [ ] Eux / GCP : confirmer la region et le projet GCP utilises.
- [ ] Eux / GCP : appliquer ou confirmer `--no-allow-unauthenticated`.
- [ ] Eux / GCP : creer/valider le service account appelant.
- [ ] Eux / GCP : donner `roles/run.invoker` au service account appelant sur le service Cloud Run.

Commande cible cote GCP :

```bash
gcloud run deploy badge-generator-service \
  --image gotenberg/gotenberg:8 \
  --region europe-west9 \
  --no-allow-unauthenticated \
  --timeout 60 \
  --min-instances 1
```

### T+5 a T+10 — generer le token OIDC cote OVH/Nest

- [ ] Moi / Rabie : recuperer l'audience a utiliser, normalement l'URL Cloud Run exacte.
- [ ] Moi / Rabie : configurer cote Nest/OVH les variables minimales :
  - `GOTENBERG_URL=<cloud-run-url>/forms/chromium/convert/html`
  - `GOTENBERG_AUDIENCE=<cloud-run-url>`
  - credential du service account appelant, sans le commiter.
- [ ] Moi / Rabie : generer un token OIDC avec audience Cloud Run.
- [ ] Moi / Rabie : ajouter le header HTTP :

```http
Authorization: Bearer <OIDC_TOKEN>
```

Point a confirmer avec eux : pour le smoke test immediat, est-ce qu'on utilise une cle service account temporaire cote OVH/Nest, ou une impersonation `gcloud` juste pour tester. Pour l'event, eviter tout secret committe et garder le stockage dans l'environnement serveur/secret.

### T+10 a T+15 — smoke test conjoint

- [ ] Test 1 : appel sans token vers Cloud Run => attendu `401` ou `403`.
- [ ] Test 2 : appel avec token OIDC valide => attendu `200`.
- [ ] Test 3 : reponse Gotenberg => `Content-Type: application/pdf`.
- [ ] Test 4 : PDF non vide et ouvrable.
- [ ] Test 5 : logs Cloud Run visibles avec statut, latence, erreur eventuelle.

Critere de sortie du creneau :

- [ ] Cloud Run Gotenberg est prive.
- [ ] OVH/Nest sait obtenir un token OIDC.
- [ ] Un appel authentifie genere un PDF.
- [ ] Un appel anonyme est refuse.
- [ ] On peut passer ensuite au branchement propre dans le worker Nest avec fallback Puppeteer.

---

## Repartition des responsabilites

| Responsable | A faire | Statut |
| --- | --- | --- |
| Eux / GCP | Deployer ou confirmer le service Cloud Run Gotenberg. | En cours |
| Eux / GCP | Configurer `--no-allow-unauthenticated`. | Priorite maintenant |
| Eux / GCP | Configurer `roles/run.invoker` pour le service account appelant. | Priorite maintenant |
| Eux / GCP | Fournir URL Cloud Run, region, projet, nom du service account. | Priorite maintenant |
| Eux / GCP | Confirmer ou non le besoin d'allowlist IP OVH en plus de l'OIDC. | A trancher |
| Moi / Rabie | Generer le token OIDC cote OVH/Nest avec la bonne audience. | Priorite maintenant |
| Moi / Rabie | Appeler `/forms/chromium/convert/html` avec le header `Authorization`. | Priorite maintenant |
| Moi / Rabie | Lancer le smoke test sans token puis avec token. | Dans 15 min |
| Moi / Rabie | Garder Puppeteer local en fallback jusqu'a validation template reel. | A faire apres smoke |
| Moi / Rabie | Documenter resultat du smoke test dans ce fichier. | A faire pendant/apres call |

---

## Fait

- [x] POC local Gotenberg realise.
- [x] Comparaison Puppeteer vs Gotenberg faite.
- [x] Architecture cible clarifiee : Cloud Run stateless, aucun flux entrant vers OVH.
- [x] Decision de principe : Cloud Run prive + OIDC.
- [x] Decision de principe : Nest/OVH construit le HTML et stocke ensuite le PDF dans R2.
- [x] Decision de principe : conserver Puppeteer local en fallback.

---

## Smoke test — commandes a adapter

Variables a renseigner pendant le call :

```bash
export GOTENBERG_BASE_URL="https://<service-cloud-run>"
export GOTENBERG_URL="$GOTENBERG_BASE_URL/forms/chromium/convert/html"
export GOTENBERG_AUDIENCE="$GOTENBERG_BASE_URL"
```

Test attendu en refus anonyme :

```bash
curl -i "$GOTENBERG_URL"
```

Attendu : `401` ou `403`.

Test attendu avec token :

Preparer un HTML minimal pour le smoke test :

```html
<!doctype html>
<html>
  <body>
    <h1>Smoke Gotenberg OK</h1>
  </body>
</html>
```

```bash
TOKEN="<OIDC_TOKEN>"
curl \
  -H "Authorization: Bearer $TOKEN" \
  -F 'files=@index.html' \
  "$GOTENBERG_URL" \
  --dump-header smoke-headers.txt \
  --output smoke-gotenberg.pdf
```

Attendu :

- statut HTTP `200`;
- fichier `smoke-gotenberg.pdf` non vide;
- PDF ouvrable;
- log Cloud Run correspondant visible.

---

## Resultat du smoke test

Date/heure :

| Test | Resultat | Note |
| --- | --- | --- |
| Appel sans token refuse | A remplir |  |
| Token OIDC genere cote OVH/Nest | A remplir |  |
| Appel avec token accepte | A remplir |  |
| PDF genere | A remplir |  |
| Logs Cloud Run visibles | A remplir |  |

Decision apres test :

- [ ] Go pour brancher le worker Nest sur Cloud Run en staging.
- [ ] Garder fallback Puppeteer actif.
- [ ] Creer les tickets restants : logs metier, template reel, mesure payload/fonts, alerte monitoring.

---

## Points de vigilance apres ce creneau

- Ne jamais commiter de cle service account.
- Verifier que l'audience du token correspond exactement a l'URL Cloud Run attendue.
- Confirmer si `europe-west9` est valide cote organisation GCP, sinon noter la region retenue.
- Rejouer le comparateur sur le template final de l'event.
- Mesurer payload HTML + assets + fonts avant bascule prod.
- Garder le fallback Puppeteer jusqu'a validation pixel/template reel.
