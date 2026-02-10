# Architecture review — Playwright (JS/TS) — Semaine 1

## Objectif
Documenter rapidement :
- l’architecture du projet (ce qui est où, pourquoi)
- les décisions visibles (tooling, auth, CI)
- les problèmes rencontrés à l’exécution + corrections
- les risques et recommandations

> Contexte : lecture + exécution locale + debug.

---

## Vue d’ensemble de l’architecture

### Structure (high-level)
- .github/
  -  actions/  (composites / réutilisables)
    - playwright-setup 
    - playwright-report 
  -  workflows/  (pipelines CI en  .yml )
-  auth/ 
  - artefacts de session (guestSession.json )
-  fixtures/ 
  -  API
    - api-request-fixture.ts  (fixtures côté API / client)
    - plain-function.ts
  - POM
    -  page-object.ts  (fixtures pour Page Object Model)
    - test-options.ts
-  pages/  (classes/pages  .ts )
  - articlesPage.ts
  - homePage.ts
  - navPage.ts
-  test-data/  (données de test  .json )
  - articleData.json
  - invalidCredentials.json
-  tests/ 
  - API (tests API) 
    - articles.spec.ts
    - authentification.spec.ts 
  - clientSide (tests UI / client side)
    - article.spec.ts
    - failingTest.spec.ts
    - home.spec.ts
    - nav.setup.ts
- fichiers de config/outillage :
  -  .gitignore 
  -  package.json 
  - config lint/format ( eslint ,  prettier )
  - hooks git ( husky , pre-commit)
  - config TS ( tsconfig )
  -  playwright.config.* 

---

## Décisions de conception notables

### 1) Tooling qualité (lint/format + hooks)
- ESLint + Prettier pour standardiser le code (JS/TS)
- Husky + (pré-commit) pour exécuter automatiquement des vérifications avant commit/push

**Pourquoi c’est un bon choix**
- réduit la dette de style
- évite de “casser le build” pour des erreurs triviales
- rend les PR plus faciles à relire

---

### 2) Auth dédiée via setup + artefact de session
On observe :
- un fichier de setup d’authentification (ex:  auth.setup.ts )
- un dossier  auth/  qui contient un artefact de session (ex:  guestSession.json )

**Hypothèse de design**
- séparer la **logique** d’authentification (setup) de la **donnée** (storage/session)
- éviter la duplication de code d’auth dans chaque test
- accélérer l’exécution et stabiliser les tests (réutilisation storageState/session)

**Question ouverte (à clarifier dans la doc)**
- pourquoi  auth/  ne contient qu’un fichier session ?
- quelles règles de renouvellement/invalidations de session ?
- quelles différences guest vs user ?

---

### 3) CI/CD via GitHub Actions
- Workflows  .yml  sous  .github/workflows 
- Possibles actions réutilisables : setup Playwright, génération/collecte de report

**Point d’attention**
- assurer une parité local/CI sur le chargement des env + navigateurs

---

## Gestion des environnements (env vars)

### Ce qui est attendu
- secrets/credentials stockés en variables d’environnement (bonne pratique sécurité)
- présence d’un  .env.example  pour documenter les variables attendues

### Problème rencontré (local)
- confusion sur le fichier env chargé par défaut (ex:  .env.dev  pris comme défaut via config)
- conséquence : URL invalide pour l’auth (via API), ou variables manquantes

**Recommandation**
- documenter explicitement :
  - quel  .env  est chargé selon le mode (local/CI)
  - comment overrider (commande / variable / flag)
- ajouter une validation au démarrage :
  - fail fast si  BASE_URL ,  API_URL ,  USER ,  PASSWORD  manquent

---

## Exécution locale — constats & debug

### Constats
- erreurs de compilation (TypeScript) lors de la première exécution
- erreur “URL invalide” durant l’auth via API
- test API : code attendu ≠ code obtenu (ex: 401 au lieu de 200)
- comportement différent selon navigateur :
  - setup/auth OK sur Chrome
  - setup/auth KO sur Firefox (et/ou autres)

### Cause identifiée (hypothèse validée par correctif)
- le projet de test (API) dépend d’un  setup  qui n’était pas déclaré / chaîné correctement pour tous les projets/navigateurs
- correction : ajout d’une dépendance explicite (ex:  dependencies: ['setup'] ) pour garantir l’ordre d’exécution
- effet : token correctement récupéré → auth API OK

---

## Risques identifiés (architecture)

### R1 — Dépendances implicites (setup/auth)
**Risque**
- des tests démarrent sans prérequis (setup non exécuté) → 401/403, flaky, pertes de temps

**Mitigation**
- définir clairement les  projects  Playwright et leurs  dependencies 
- documenter “setup first” (local & CI)

---

### R2 — Parité navigateurs (Chrome vs Firefox)
**Risque**
- tests qui passent “par chance” sur un navigateur, mais échouent ailleurs
- faux sentiment de couverture multi-browser

**Mitigation**
- s’assurer que la config/setup est appliquée à tous les navigateurs ciblés
- ajouter une étape CI “smoke multi-browser” minimale

---

### R3 — Chargement env non explicite
**Risque**
- erreurs difficiles à diagnostiquer (URL invalides, credentials vides)
- divergence local vs CI

**Mitigation**
- standardiser le chargement env ( dotenv , convention de nommage)
- fail fast sur variables manquantes
- README : “Local run” + “CI run” + exemples

---

### R4 — Artefacts de session (guestSession.json)
**Risque**
- session périmée/invalidée → tests instables
- risque sécurité si un storageState “réel” est commité

**Mitigation**
- vérifier que le storageState commité est non-sensible (guest only)
- stratégie de refresh (re-génération automatique)
- clarifier si  guestSession.json  est un artefact “exemple” ou “source de vérité”

---

## Recommandations (actionnables)

1) README : section “Auth & setup”
- pourquoi  auth.setup.ts 
- comment est généré/utilisé  guestSession.json 
- comment régénérer la session

2) README : section “Environment variables”
- liste des variables attendues
- exemple  .env.example 
- commande(s) pour lancer localement

3) Playwright config
- rendre explicites les  projects  et  dependencies 
- garantir la cohérence multi-browser

4) Qualité d’exécution
- conserver traces/reporting en CI
- documenter où trouver les artefacts (report, traces, vidéos)

---

## Notes pour contribution (idées PR/issue)
- Issue : “Env loading ambiguity leads to invalid auth URL in local run”
- Issue : “Setup dependency not applied across all browsers/projects (401 vs 200)”
- PR doc : clarifier auth + env + multi-browser expectations
