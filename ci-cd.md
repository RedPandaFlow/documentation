# Pipeline CI/CD

## Choix de la plateforme

Le projet est hébergé sur une **organisation GitHub** (`RedPandaFlow`). Le
pipeline est donc réalisé avec **GitHub Actions** plutôt que GitLab CI : nous
n'avons pas d'accès à une instance GitLab ni à la plateforme Scalingo dans le
cadre de ce projet. Les principes restent identiques (build, tests, publication
d'image dans un registre) ; seuls les outils changent.

Pour la même raison, **le déploiement automatisé n'est pas mis en place** : il
nécessiterait un accès à Scalingo (ou une autre plateforme) et les identifiants
associés. Le reste de la chaîne (intégration continue + publication d'image) est
en place.

## Pipeline backend

Fichier : `redpandaflow-backend/.github/workflows/ci.yml`.

### Déclenchement

- `push` sur `dev` et `main`
- `pull_request` vers `dev` et `main`

Un mécanisme de `concurrency` annule un run précédent encore en cours sur la
même branche.

### Étapes (jobs)

| Job                    | Rôle                                                                       |
| ---------------------- | -------------------------------------------------------------------------- |
| **Secret scan**        | Scan de l'historique git avec `gitleaks` ; échoue si un secret est détecté |
| **Build (.NET)**       | `dotnet restore` + `dotnet build -c Release` de l'API                      |
| **Test (.NET)**        | `dotnet test` du projet xUnit (`tests/RedPandaFlow.Tests`)                 |
| **Build & push image** | Construit l'image Docker et la publie sur le registre                      |

Le job d'image dépend de la réussite de `Build` **et** `Test`
(`needs: [build, test]`).

### Tests automatisés

Le projet `RedPandaFlow.Tests` (xUnit) couvre de la logique pure, sans base de
données :

- `ServiceResult<T>` : fabrique des résultats `Ok` / `Fail`.
- `JwtTokenService` : génération du jeton d'accès (claims utilisateur), du jeton
  de rafraîchissement, et relecture d'un jeton.

### Registre d'images

Les images sont publiées sur le **GitHub Container Registry (ghcr.io)**,
équivalent GitHub du GitLab Container Registry :

```bash
ghcr.io/redpandaflow/redpandaflow-backend
```

Tags générés automatiquement (via `docker/metadata-action`) :

- nom de la branche (ex. `dev`)
- SHA du commit
- `latest` (uniquement sur `main`)

La publication se fait sur un `push` ou sur une pull request après la **construction** (validation) 
de l'image vers `dev`/`main`. Le job d'image est configuré pour ne pas s'exécuter sur les autres branches.

### Sécurité du pipeline

- `permissions: contents: read` au niveau du workflow (principe du moindre
  privilège) ; seul le job d'image obtient `packages: write`.
- Authentification au registre via le `GITHUB_TOKEN` intégré (aucun secret
  personnel stocké).
- Aucun secret ni port sensible en clair dans le workflow ; scan `gitleaks`
  pour empêcher toute fuite.

## Pipeline frontend

Fichier : `redpandaflow-frontend/.github/workflows/ci.yml`.

### Déclenchement

- `push` sur `dev` et `main`
- `pull_request` vers `dev` et `main`

Un mécanisme de `concurrency` annule un run précédent encore en cours sur la même branche.

### Étapes (jobs)

| Job                    | Rôle                                                                       |
| ---------------------- | -------------------------------------------------------------------------- |
| **Secret scan**        | Scan de l'historique git avec `gitleaks` ; échoue si un secret est détecté |
| **Build & Lint**       | Construit l'application et exécute le linting pour faire les tests         |
| **Docker Build**       | Construit l'image Docker et la publie sur le registre                      |

Le job d'image dépend de la réussite de `Secret scan` **et** `Build & Lint`
(`needs: [secret-scan, build-and-lint]`).

### Tests automatisés

Les tests sont réalisés via le linting (ESLint) : il vérifie la syntaxe, les
règles de style, et peut aussi détecter des erreurs potentielles (ex. variable
non utilisée). Il n'y a pas de tests unitaires formels, mais le linting assure 
une validation de base du code.

### Registre d'images

Les images sont publiées sur le **GitHub Container Registry (ghcr.io)**,
équivalent GitHub du GitLab Container Registry :

```bash
ghcr.io/redpandaflow/redpandaflow-frontend
```

Tags générés automatiquement (via `docker/metadata-action`) :

- nom de la branche (ex. `dev`)
- SHA du commit
- `latest` (uniquement sur `main`)

La publication se fait sur un `push` ou sur une pull request après la **construction** (validation) 
de l'image vers `dev`/`main`. Le job d'image est configuré pour ne pas s'exécuter sur les autres branches.

### Sécurité du pipeline

- `permissions: contents: read` au niveau du workflow (principe du moindre
  privilège) ; seul le job d'image obtient `packages: write`.
- Authentification au registre via le `GITHUB_TOKEN` intégré (aucun secret
  personnel stocké).
- Aucun secret ni port sensible en clair dans le workflow ; scan `gitleaks`
  pour empêcher toute fuite.

## Protection des branches

Des _rulesets_ GitHub protègent `main` et `dev` : pull request obligatoire,
blocage du force-push, et passage obligatoire des checks de la CI avant fusion.
