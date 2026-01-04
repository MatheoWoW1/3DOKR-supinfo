# Application de Vote - Projet Dockerisé

## Description du Projet

Application distribuée permettant à une audience de voter entre deux options. L'application est composée de plusieurs services qui communiquent entre eux via Docker.

### Architecture

- **vote** : Application web Python (Flask) permettant de voter
- **result** : Application web Node.js affichant les résultats en temps réel
- **worker** : Service .NET qui transfère les votes de Redis vers PostgreSQL
- **redis** : Base de données Redis (file d'attente des votes)
- **db** : Base de données PostgreSQL (stockage persistant)

### Schéma d'Architecture

```
┌─────────┐      ┌───────┐      ┌────────┐      ┌──────┐      ┌────────┐
│  Vote   │─────>│ Redis │─────>│ Worker │─────>│  DB  │<─────│ Result │
│ (Flask) │      │       │      │ (.NET) │      │ (PG) │      │(Node.js)│
└─────────┘      └───────┘      └────────┘      └──────┘      └────────┘
   :5000                                          (volume)         :5001
```

## Prérequis

- Docker
- Docker Compose

## Lancement de l'Application

### 1. Construction et démarrage

```bash
docker compose up --build
```

L'option `--build` reconstruit les images si des modifications ont été apportées au code.

### 2. Accès aux applications

- **Application de vote** : http://localhost:5000
- **Application de résultats** : http://localhost:5001

### 3. Arrêt de l'application

```bash
docker compose down
```

Pour supprimer également les volumes (données) :

```bash
docker compose down -v
```

## Fonctionnalités Implémentées

### Dockerfiles

Chaque module possède son propre Dockerfile respectant les bonnes pratiques :

- **vote** : Image Python 3.11 slim
- **worker** : Build multi-stage avec .NET 7
- **result** : Image Node.js 18 Alpine

### Docker Compose

Le fichier `docker-compose.yml` définit :

- ✅ **Dépendances** : Les services démarrent dans le bon ordre grâce à `depends_on` avec `condition: service_healthy`
- ✅ **Health checks** : Redis et PostgreSQL sont surveillés pour garantir leur disponibilité
- ✅ **Volumes** : Les données PostgreSQL sont persistées dans un volume Docker
- ✅ **Réseaux** : Isolation entre frontend et backend
- ✅ **Restart policy** : Les conteneurs redémarrent automatiquement en cas d'échec

### Réseaux

Deux réseaux isolés :

- **frontend** : Accessible par les applications web (vote et result)
- **backend** : Communications internes (worker, redis, db)

### Persistance des Données

Les votes sont sauvegardés dans PostgreSQL via le volume `db-data`. Les données ne sont pas perdues au redémarrage.

## Déploiement sur Docker Swarm

Pour déployer l'application sur un cluster Docker Swarm, consultez le fichier [SWARM_DEPLOYMENT.md](SWARM_DEPLOYMENT.md).

## Modifications Apportées

Les scripts bash originaux ont été remplacés par Docker et Docker Compose pour :

- Simplifier le déploiement
- Garantir la portabilité
- Automatiser la gestion des dépendances
- Assurer la persistance des données

## Technologies Utilisées

- Python 3.11 (Flask)
- Node.js 18
- .NET Core 7
- PostgreSQL 15
- Redis 7
- Docker & Docker Compose

## Auteur

Projet réalisé dans le cadre du cours de conteneurisation - 3ème année BUT Informatique
