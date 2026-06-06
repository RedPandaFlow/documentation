# Conteneurisation

Ce document justifie les choix de conteneurisation du projet : rôle de chaque
conteneur, contenu des Dockerfiles, organisation du réseau, gestion des volumes
et bonnes pratiques appliquées.

## Justification des choix

Le projet est conteneurisé dès la phase de développement pour garantir un
environnement **identique pour toute l'équipe** et **reproductible** (même
versions de runtime, de base de données et de configuration). Tout se lance
avec une seule commande `docker compose up`.

L'architecture suit le minimum imposé (API, interface web, base de données)
auquel s'ajoute un service supplémentaire, **pgAdmin**, pour administrer la base
sans l'exposer publiquement.

## Les conteneurs

| Conteneur  | Image de base                                                                                 | Rôle                       |
| ---------- | --------------------------------------------------------------------------------------------- | -------------------------- |
| `backend`  | build : `mcr.microsoft.com/dotnet/sdk:10.0`, runtime : `mcr.microsoft.com/dotnet/aspnet:10.0` | API REST + SignalR         |
| `frontend` | build : `node:26-alpine`, runtime : `nginxinc/nginx-unprivileged:stable-alpine`               | SPA React servie par nginx |
| `db`       | `postgres:16-alpine`                                                                          | Base de données            |
| `pgadmin`  | `dpage/pgadmin4`                                                                              | Administration de la base  |

## Dockerfiles

### Backend (`redpandaflow-backend/Dockerfile`)

Build **multi-étapes** :

1. **Étape `build`** (SDK .NET 10) : restauration des paquets puis
   `dotnet publish -c Release` de l'API. Les `.csproj` sont copiés et restaurés
   avant le reste du code pour profiter du cache de couches Docker.
2. **Étape runtime** (image `aspnet`, sans le SDK) : on ne copie que le résultat
   publié → image finale réduite (~264 Mo).

Bonnes pratiques :

- `--no-restore` et `/p:UseAppHost=false` à la publication (build plus léger).
- Exécution en **utilisateur non-root** via `USER $APP_UID` (uid 1654, fourni par
  l'image .NET).
- Contexte de build **local au dépôt** (chemins `src/...`), ce qui permet de
  construire l'image aussi bien via le compose qu'en CI sur le dépôt seul.

### Frontend (`redpandaflow-frontend/Dockerfile`)

Build **multi-étapes** :

1. **Étape `builder`** (`node:26-alpine`) : `npm ci` (installation reproductible
   à partir du `package-lock.json`) puis `npm run build` (Vite).
2. **Étape runtime** (`nginx-unprivileged`) : on ne copie que le `dist/` compilé.
   Image finale ~55 Mo.

Bonnes pratiques :

- Image **nginx non-privilégiée** : nginx tourne en utilisateur non-root (uid 101).
- Le port d'écoute de nginx est injecté au démarrage via le mécanisme de
  templates de l'image (`envsubst` sur `FRONTEND_PORT`), avec un filtre
  (`NGINX_ENVSUBST_FILTER`) pour ne substituer que cette variable.

### Base de données

PostgreSQL utilise directement l'**image officielle** `postgres:16-alpine` : pas
de Dockerfile dédié, conformément aux bonnes pratiques (on ne recompile pas une
image standard).

## Organisation du réseau

Deux réseaux Docker isolent les couches (voir [architecture.md](architecture.md)) :

- `frontend-network` : `frontend` ↔ `backend`
- `backend-network` : `backend` ↔ `db` ↔ `pgadmin`

La base de données n'est accessible que sur `backend-network`. Le `backend`,
présent sur les deux réseaux, est le seul pont entre présentation et données.

## Gestion des volumes et des données

- **`postgres_data`** : fichiers de la base. Les données survivent à
  `docker compose down` (seul `down -v` les supprime).
- **`pgadmin_data`** : configuration de pgAdmin.

Séparation nette entre **données applicatives** (volumes nommés) et
**conteneurs** (éphémères, reconstructibles à tout moment).

### Sauvegarde et restauration

Deux scripts dans `redpandaflow-infra/scripts/` opèrent sur le conteneur `db` en
cours d'exécution (ils lisent les identifiants directement dans le conteneur) :

```bash
./scripts/backup.sh                          # dump horodaté dans backups/
./scripts/restore.sh backups/<fichier>.dump  # restauration
```

Le format `pg_dump -Fc` (custom) permet une restauration avec
`pg_restore --clean --if-exists`.
