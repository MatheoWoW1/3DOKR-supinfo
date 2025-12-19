# Voting App – Dockerisée

## Lancement en local
docker compose up --build

- Vote : http://localhost:5000
- Result : http://localhost:5001

## Arrêt
docker compose down

## Données persistantes
Les votes sont stockés dans PostgreSQL via un volume Docker.
