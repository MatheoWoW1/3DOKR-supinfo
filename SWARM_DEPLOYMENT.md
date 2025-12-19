# Déploiement Docker Swarm

## 1. Création du cluster

Sur le manager :
docker swarm init --advertise-addr <IP_MANAGER>

Sur les workers :
docker swarm join --token <TOKEN> <IP_MANAGER>:2377

## 2. Construction des images

docker build -t voting-app_vote ./vote
docker build -t voting-app_result ./result
docker build -t voting-app_worker ./worker

## 3. Déploiement de la stack

docker stack deploy -c docker-compose.swarm.yml voting

## 4. Vérification

docker service ls
docker service ps voting_vote

## 5. Tolérance aux pannes

Les services vote et result sont répliqués sur plusieurs nœuds.
Si un worker tombe, Swarm redéploie automatiquement les conteneurs.
