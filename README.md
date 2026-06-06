# Documentation RedPandaFlow

RedPandaFlow est une application de kanban collaboratif (inspirée de Trello)
permettant à des équipes d'organiser leur travail en espaces de travail,
tableaux, colonnes et cartes, avec une synchronisation en temps réel entre
collaborateurs.

Ce dépôt centralise la documentation transverse du projet : architecture,
conteneurisation, pipeline CI/CD et organisation de l'équipe. La documentation
propre à chaque service se trouve dans le README de son dépôt.

## Sommaire

- [Architecture du projet](architecture.md) — schéma, services et communication
- [Conteneurisation](conteneurisation.md) — choix Docker, Dockerfiles, réseau, volumes, bonnes pratiques
- [Pipeline CI/CD](ci-cd.md) — GitHub Actions, tests, registre d'images
- [Authentification JWT](authentification-jwt.md) — fonctionnement de l'authentification
- [Équipe et répartition](equipe.md) — membres et organisation

## Dépôts

- [redpandaflow-backend](https://github.com/RedPandaFlow/redpandaflow-backend) — API ASP.NET Core (.NET 10), Clean Architecture, EF Core + PostgreSQL, SignalR
- [redpandaflow-frontend](https://github.com/RedPandaFlow/redpandaflow-frontend) — React 19, Vite, Tailwind CSS, servi par nginx
- [redpandaflow-infra](https://github.com/RedPandaFlow/redpandaflow-infra) — stack docker-compose pour lancer tout le projet
- [documentation](https://github.com/RedPandaFlow/documentation) — ce dépôt

## Fonctionnalités principales

- Espaces de travail, tableaux, colonnes et cartes avec glisser-déposer
- Synchronisation temps réel (présence, mutations de tableau, notifications)
- Commentaires, étiquettes, checklists et assignations sur les cartes
- Historique d'activité par carte
- Notifications avec badge de non-lus
- Recherche transverse (espaces, tableaux, cartes)
- Profils utilisateurs avec avatar, suppression de compte
- Rôles sur les espaces et tableaux (Admin, Membre, Lecteur)
