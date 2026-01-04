# Déploiement de l'Application sur Docker Swarm

Ce document décrit le processus complet de déploiement de l'application de vote sur un cluster Docker Swarm composé de **1 nœud manager** et **2 nœuds worker**.

## Architecture du Cluster

```
┌─────────────────┐
│  Manager Node   │ (Orchestration + Services)
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───┴───┐ ┌──┴────┐
│Worker1│ │Worker2│ (Exécution des services)
└───────┘ └───────┘
```

## Prérequis

- 3 machines (physiques ou virtuelles) avec Docker installé
- Connectivité réseau entre les 3 machines
- Ports ouverts : 2377 (cluster), 7946 (communication), 4789 (overlay network)

## Étape 1 : Initialisation du Cluster Swarm

### Sur le nœud Manager

```bash
# Initialiser le cluster Swarm
docker swarm init --advertise-addr <IP_DU_MANAGER>
```

Cette commande affiche deux tokens :
- Un token pour ajouter des managers
- Un token pour ajouter des workers

**Exemple de sortie :**
```
Swarm initialized: current node (xyz123) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-xxxxx <IP_DU_MANAGER>:2377
```

Notez bien le token et l'IP pour l'étape suivante.

### Sur chaque nœud Worker

Exécutez la commande fournie par le manager :

```bash
docker swarm join --token SWMTKN-1-xxxxx <IP_DU_MANAGER>:2377
```

### Vérification du cluster

Sur le manager, vérifiez que tous les nœuds sont bien connectés :

```bash
docker node ls
```

Vous devriez voir 3 nœuds : 1 manager et 2 workers.

## Étape 2 : Construction des Images Docker

Sur le nœud manager, depuis le répertoire du projet :

```bash
# Construction de l'image vote
docker build -t voting-app_vote ./vote

# Construction de l'image result
docker build -t voting-app_result ./result

# Construction de l'image worker
docker build -t voting-app_worker ./worker
```

### Option : Utilisation d'un registre Docker

Pour un déploiement sur plusieurs nœuds, il est recommandé d'utiliser un registre Docker :

```bash
# Lancer un registre local
docker service create --name registry --publish 5000:5000 registry:2

# Tag et push des images
docker tag voting-app_vote localhost:5000/voting-app_vote
docker push localhost:5000/voting-app_vote

docker tag voting-app_result localhost:5000/voting-app_result
docker push localhost:5000/voting-app_result

docker tag voting-app_worker localhost:5000/voting-app_worker
docker push localhost:5000/voting-app_worker
```

Si vous utilisez un registre, modifiez les noms d'images dans `docker-compose.swarm.yml`.

## Étape 3 : Déploiement de la Stack

Sur le nœud manager :

```bash
docker stack deploy -c docker-compose.swarm.yml voting
```

Cette commande déploie tous les services définis dans le fichier de configuration.

## Étape 4 : Vérification du Déploiement

### Lister les services de la stack

```bash
docker stack services voting
```

Vous devriez voir 5 services :
- voting_vote (2 réplicas)
- voting_result (2 réplicas)
- voting_worker (1 réplica)
- voting_redis (1 réplica)
- voting_db (1 réplica)

### Voir les tâches d'un service

```bash
docker service ps voting_vote
```

Cette commande affiche sur quels nœuds les conteneurs sont déployés.

### Consulter les logs d'un service

```bash
docker service logs voting_vote
docker service logs voting_worker
```

## Étape 5 : Accès aux Applications

Les applications sont accessibles sur n'importe quel nœud du cluster :

- **Application de vote** : http://<IP_MANAGER>:5000
- **Application de résultats** : http://<IP_MANAGER>:5001

Grâce au routing mesh de Docker Swarm, vous pouvez aussi accéder aux applications via l'IP des workers.

## Caractéristiques du Déploiement

### Réplication et Haute Disponibilité

- **vote** : 2 réplicas répartis sur les nœuds
- **result** : 2 réplicas répartis sur les nœuds
- **worker** : 1 réplica (stateless)
- **redis** et **db** : 1 réplica sur le manager (stateful)

### Tolérance aux Pannes

Si un nœud worker tombe :
1. Docker Swarm détecte la panne
2. Les services sur ce nœud sont automatiquement redéployés sur les nœuds disponibles
3. Les applications restent accessibles grâce aux réplicas multiples

**Test de tolérance aux pannes :**

```bash
# Sur un nœud worker
docker swarm leave

# Sur le manager, vérifier le redéploiement
docker service ps voting_vote
```

Les conteneurs sont automatiquement recréés sur les nœuds restants.

### Stratégie de Mise à Jour

Les services vote et result utilisent une stratégie de rolling update :
- `parallelism: 1` - Mise à jour d'un conteneur à la fois
- `delay: 10s` - Délai de 10 secondes entre chaque mise à jour

Pour mettre à jour un service :

```bash
docker service update --image voting-app_vote:v2 voting_vote
```

## Gestion de la Stack

### Mettre à l'échelle un service

```bash
docker service scale voting_vote=3
```

### Supprimer la stack

```bash
docker stack rm voting
```

### Quitter le cluster (worker)

```bash
docker swarm leave
```

### Détruire le cluster (manager)

```bash
docker swarm leave --force
```

## Réseaux

Le fichier `docker-compose.swarm.yml` définit deux réseaux overlay :

- **frontend** : Communication entre vote/result et les utilisateurs
- **backend** : Communication interne entre worker, redis et db

Les réseaux overlay permettent la communication entre conteneurs sur différents nœuds.

## Volumes et Persistance

Le volume `db-data` assure la persistance des données PostgreSQL. En production, il est recommandé d'utiliser :
- Un driver de volume distribué (comme REX-Ray)
- Un placement contraint sur le manager
- Des sauvegardes régulières

## Troubleshooting

### Les services ne démarrent pas

```bash
docker service ps voting_vote --no-trunc
```

### Problèmes de réseau

```bash
docker network ls
docker network inspect voting_frontend
```

### Réinitialiser un nœud

```bash
docker swarm leave --force
docker system prune -a
```

## Différences avec Docker Compose

| Fonctionnalité | Docker Compose | Docker Swarm |
|----------------|----------------|--------------|
| `build` | ✅ Supporté | ❌ Pas supporté |
| `depends_on` | ✅ Supporté | ❌ Ignoré |
| `healthcheck` | ✅ Supporté | ⚠️ Limité |
| `deploy` | ❌ Ignoré | ✅ Supporté |
| Réplication | ❌ Non | ✅ Oui |
| Rolling updates | ❌ Non | ✅ Oui |

C'est pourquoi nous avons deux fichiers séparés : `docker-compose.yml` pour le développement local et `docker-compose.swarm.yml` pour la production en cluster.

## Conclusion

Ce processus de déploiement permet :
- ✅ Une haute disponibilité des applications web
- ✅ Une tolérance aux pannes d'un nœud worker
- ✅ Une mise à l'échelle facile des services
- ✅ Une gestion centralisée depuis le nœud manager
