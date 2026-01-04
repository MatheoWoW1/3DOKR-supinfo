# Guide de Préparation du Rendu

## Création de l'archive ZIP pour le rendu

### Depuis le terminal Linux

```bash
# Se placer dans le répertoire parent
cd /home/JeremyDebian/projet_ecole/projet_groupe_Supinfo/

# Créer l'archive ZIP
zip -r projet_vote_docker.zip 3DOKR-supinfo/ -x "3DOKR-supinfo/.git/*" "3DOKR-supinfo/**/node_modules/*" "3DOKR-supinfo/**/__pycache__/*"
```

### Contenu de l'archive

L'archive doit contenir :
- ✅ Tous les Dockerfiles (vote, worker, result)
- ✅ docker-compose.yml
- ✅ docker-compose.swarm.yml
- ✅ README.md (instructions de lancement)
- ✅ SWARM_DEPLOYMENT.md (guide de déploiement Swarm)
- ✅ MODIFICATIONS.md (journal des modifications)
- ✅ Code source de tous les modules (vote, worker, result)
- ✅ Fichiers de dépendances (requirements.txt, package.json, etc.)

### Vérification avant envoi

1. **Tester le projet localement :**
```bash
cd 3DOKR-supinfo
docker compose up --build
# Vérifier http://localhost:5000 et http://localhost:5001
docker compose down
```

2. **Vérifier la taille de l'archive :**
```bash
ls -lh projet_vote_docker.zip
```
L'archive devrait faire moins de 10 Mo.

3. **Vérifier le contenu de l'archive :**
```bash
unzip -l projet_vote_docker.zip | less
```

### Documents à consulter par le correcteur

1. **README.md** : Pour lancer le projet en local avec Docker Compose
2. **SWARM_DEPLOYMENT.md** : Pour déployer sur un cluster Docker Swarm
3. **MODIFICATIONS.md** : Pour comprendre toutes les modifications apportées

## Rappel des URLs

- Application de vote : http://localhost:5000
- Application de résultats : http://localhost:5001

## Commandes utiles

```bash
# Lancer l'application
docker compose up --build

# Arrêter l'application
docker compose down

# Supprimer les volumes (réinitialiser les données)
docker compose down -v

# Voir les logs
docker compose logs -f

# Voir l'état des conteneurs
docker compose ps
```