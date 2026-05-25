# Authentification JWT

Ce document explique comment fonctionne l'authentification dans RedPandaFlow.

## Le principe

Quand un utilisateur se connecte, le serveur va lui remettre deux passes
numériques (des "jetons"). À chaque action de l'utilisateur, son navigateur
présente ces jetons et le serveur vérifie qu'ils sont valides avant
d'autoriser l'action.

Aucun mot de passe ne circule entre le navigateur et le serveur après la
connexion. Seuls les jetons sont échangés.

## Les deux jetons

L'application utilise deux jetons aux rôles différents.

**Le jeton d'accès** est utilisé pour prouver l'identité à chaque requête.
Il a une durée de vie courte (quinze minutes). S'il est volé, le voleur n'a
qu'une fenêtre très limitée pour s'en servir.

**Le jeton de rafraîchissement** sert uniquement à obtenir un nouveau jeton
d'accès quand l'ancien expire. Il a une durée de vie plus longue (sept
jours), mais il n'est utilisable que sur un seul endpoint dédié.

Ce système permet à l'utilisateur de rester connecté plusieurs jours sans
ressaisir son mot de passe, tout en limitant les dégâts en cas de vol.

## Où sont stockés les jetons

Les deux jetons sont placés dans des cookies dits "HttpOnly". Cela
signifie simplement que le code JavaScript de la page n'a pas le droit de
les lire. Seul le navigateur sait qu'ils existent et les renvoie
automatiquement au serveur à chaque requête.

Cela protège contre une attaques très répandues (injection de
script malveillant dans la page) qui chercherait à voler les jetons.

L'application ne stocke jamais les jetons dans le stockage local du
navigateur, ni dans une variable JavaScript.

## Le cycle de vie d'une session

1. L'utilisateur s'inscrit ou se connecte avec son email et son mot de
   passe. Le serveur vérifie les identifiants et dépose les deux cookies
   dans la réponse.
2. À chaque page visitée ou action effectuée, le navigateur joint
   automatiquement le jeton d'accès. Le serveur vérifie sa signature et
   autorise l'action.
3. Quand le jeton d'accès expire (au bout de quinze minutes), la requête
   suivante échoue avec un code d'erreur. Le navigateur le détecte, appelle
   automatiquement le endpoint de rafraîchissement, obtient un nouveau
   jeton d'accès et un nouveau jeton de rafraîchissement, puis rejoue la
   requête initiale. L'utilisateur ne voit rien de tout cela.
4. Quand l'utilisateur clique sur "Se déconnecter", le serveur invalide les
   jetons et vide les cookies. Une nouvelle connexion sera nécessaire.

## Protections supplémentaires

**Rotation des jetons de rafraîchissement.** À chaque rafraîchissement, le
jeton précédent est marqué comme utilisé et remplacé par un nouveau. Un
même jeton de rafraîchissement ne peut servir qu'une seule fois.

**Détection de réutilisation.** Si un ancien jeton de rafraîchissement est
présenté une seconde fois (signe qu'il a été volé et que deux personnes
essaient de s'en servir), tous les jetons actifs de l'utilisateur sont
révoqués. La session est alors fermée partout, par sécurité.

**Hachage des mots de passe.** Le mot de passe n'est jamais stocké en
clair dans la base de données. Seule une empreinte calculée avec
l'algorithme bcrypt est conservée. Même si la base de données venait à
fuiter, les mots de passe resteraient protégés.

**Limitation du nombre de tentatives.** Les endpoints de connexion,
d'inscription et de rafraîchissement sont limités à cinq requêtes par
minute et par adresse IP, pour décourager les tentatives automatisées.

## Liste des endpoints concernés

| Endpoint                   | Rôle                                          |
| -------------------------- | --------------------------------------------- |
| `POST /api/auth/register`  | Créer un nouveau compte                       |
| `POST /api/auth/login`     | Se connecter avec email + mot de passe        |
| `POST /api/auth/refresh`   | Obtenir un nouveau jeton d'accès              |
| `GET  /api/auth/me`        | Récupérer le profil de l'utilisateur connecté |
| `POST /api/auth/logout`    | Se déconnecter et invalider les jetons        |
| `DELETE /api/auth/account` | Supprimer définitivement son compte           |
