# Guide de Soutenance - Projet Conteneurisation

**Durée : 15 min présentation + 10 min questions**

---

## 📋 Plan de la Soutenance

### Timing Recommandé
- **Introduction** : 2 minutes
- **Architecture & Choix Techniques** : 3 minutes
- **Dockerfiles** : 3 minutes
- **Docker Compose** : 3 minutes
- **Docker Swarm** : 2 minutes
- **Démonstration Live** : 2 minutes
- **Conclusion** : 1 minute

---

## 🎯 Slide 1 : Page de Titre (30 secondes)

**À l'écran :**
```
Conteneurisation d'une Application de Vote
Projet Docker & Docker Swarm

[Votre Nom]
3ème année BUT Informatique
[Date]
```

**Ce que vous dites :**
> "Bonjour, je vais vous présenter mon projet de conteneurisation d'une application distribuée de vote. Ce projet consistait à moderniser le déploiement d'une application multi-services en utilisant Docker et Docker Swarm."

---

## 🎯 Slide 2 : Contexte & Objectifs (1 min 30)

**À l'écran :**
```
Contexte Initial
- Application distribuée avec 5 composants
- Déploiement via 5 scripts bash séparés
- Pas de gestion des dépendances
- Pas de persistance des données

Objectifs du Projet
✅ Conteneuriser les 3 modules applicatifs
✅ Orchestrer avec Docker Compose
✅ Déployer sur cluster Docker Swarm (1 manager + 2 workers)
✅ Garantir haute disponibilité et persistance
```

**Ce que vous dites :**
> "À l'origine, l'application nécessitait l'exécution manuelle de 5 scripts bash dans des terminaux différents, sans garantie sur l'ordre de démarrage ni sur la persistance des données. Mon objectif était de moderniser ce déploiement en conteneurisant chaque service et en automatisant l'orchestration avec Docker Compose, puis de préparer un déploiement production sur un cluster Docker Swarm."

**💡 Conseil aisance oratoire :** Parlez avec assurance, regardez l'examinateur, pas vos notes.

---

## 🎯 Slide 3 : Architecture de l'Application (2 min)

**À l'écran :**
```
┌─────────┐      ┌───────┐      ┌────────┐      ┌──────┐      ┌────────┐
│  Vote   │─────>│ Redis │─────>│ Worker │─────>│  DB  │<─────│ Result │
│ Flask   │      │       │      │ .NET   │      │ PG   │      │Node.js │
│ Python  │      │       │      │        │      │      │      │        │
└─────────┘      └───────┘      └────────┘      └──────┘      └────────┘
   :5000                                          Volume          :5001

Flux de données :
1. Utilisateur vote via l'interface Flask
2. Vote stocké temporairement dans Redis (queue)
3. Worker .NET récupère les votes de Redis
4. Worker insère les votes dans PostgreSQL
5. Interface Node.js affiche les résultats en temps réel
```

**Ce que vous dites :**
> "L'architecture est composée de 5 services. L'utilisateur vote via une interface Python Flask sur le port 5000. Le vote est placé dans une file Redis. Un worker en .NET consomme cette file et persiste les votes dans PostgreSQL. Enfin, une interface Node.js sur le port 5001 affiche les résultats en temps réel depuis la base de données."

**💡 Points à souligner :**
- Architecture microservices découplée
- Chaque service a une responsabilité unique
- Communication asynchrone via Redis

---

## 🎯 Slide 4 : Choix Techniques - Dockerfiles (3 min)

**À l'écran :**
```
Bonnes Pratiques Appliquées

Vote (Python)
✅ Image python:3.11-slim (légère, 150MB vs 1GB)
✅ COPY des requirements.txt avant le code (cache Docker)
✅ pip install --no-cache-dir (réduction taille)

Worker (.NET)
✅ Build multi-stage (SDK 7.0 → Runtime 7.0)
✅ Image finale 4x plus petite
✅ Compilation en mode Release

Result (Node.js)
✅ Image node:18-alpine (20MB de base)
✅ npm install --production (pas de devDependencies)
✅ COPY package.json séparément (optimisation cache)
```

**Montrez le code à l'écran :**

**vote/Dockerfile :**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python", "app.py"]
```

**Ce que vous dites :**
> "Pour le module vote, j'ai utilisé une image Python slim qui est 6 fois plus légère que l'image standard. J'ai séparé la copie du requirements.txt du reste du code pour profiter du cache Docker : si je modifie juste le code, les dépendances ne sont pas réinstallées."

**worker/Dockerfile (montrez-le) :**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/runtime:7.0
WORKDIR /app
COPY --from=build /app .
CMD ["dotnet", "Worker.dll"]
```

**Ce que vous dites :**
> "Pour le worker .NET, j'ai utilisé un build multi-stage. La première étape compile avec le SDK qui fait 800MB, mais l'image finale utilise uniquement le runtime qui fait 200MB. Ça réduit considérablement la taille de l'image déployée."

**💡 Argumentation :** Insistez sur les gains (temps, espace, sécurité).

---

## 🎯 Slide 5 : Docker Compose - Orchestration (3 min)

**À l'écran :**
```
Docker Compose - Fonctionnalités Clés

1. Gestion des Dépendances
   depends_on + condition: service_healthy

2. Health Checks
   - Redis: redis-cli ping
   - PostgreSQL: pg_isready

3. Persistance
   Volume db-data → /var/lib/postgresql/data

4. Isolation Réseau
   - frontend: vote, result (public)
   - backend: worker, redis, db (interne)

5. Restart Policy
   unless-stopped
```

**Montrez le code (docker-compose.yml) :**
```yaml
db:
  image: postgres:15
  environment:
    POSTGRES_USER: vote
    POSTGRES_PASSWORD: vote
    POSTGRES_DB: votedb
  volumes:
    - db-data:/var/lib/postgresql/data
  networks:
    - backend
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U vote -d votedb"]
    interval: 5s
    timeout: 3s
    retries: 5
  restart: unless-stopped

worker:
  build: ./worker
  depends_on:
    redis:
      condition: service_healthy
    db:
      condition: service_healthy
  networks:
    - backend
  restart: unless-stopped
```

**Ce que vous dites :**
> "Docker Compose orchestre les 5 services. J'ai configuré des dépendances strictes : le worker ne démarre que si Redis et PostgreSQL sont 'healthy', pas juste démarrés. Les health checks vérifient toutes les 5 secondes que les services répondent correctement. Pour la persistance, j'utilise un volume nommé qui survit aux redémarrages. Enfin, j'ai créé deux réseaux isolés : frontend pour les applications web accessibles de l'extérieur, et backend pour les communications internes entre worker, Redis et PostgreSQL."

**💡 Pertinence :** Expliquez pourquoi c'est important (fiabilité, sécurité).

---

## 🎯 Slide 6 : Docker Swarm - Production (2 min)

**À l'écran :**
```
Déploiement en Cluster Swarm

Architecture:
┌──────────┐
│ Manager  │ (Orchestration + Redis + PostgreSQL)
└────┬─────┘
     │
  ┌──┴──┐
  │     │
┌─┴──┐ ┌┴───┐
│Wrk1│ │Wrk2│ (Applications répliquées)
└────┘ └────┘

Haute Disponibilité:
- vote: 2 réplicas
- result: 2 réplicas
- Réseaux overlay
- Rolling updates (1 conteneur à la fois, délai 10s)
- Redémarrage automatique en cas de panne
```

**Montrez le code (docker-compose.swarm.yml) :**
```yaml
vote:
  image: voting-app_vote
  ports:
    - "5000:8080"
  networks:
    - frontend
    - backend
  deploy:
    replicas: 2
    restart_policy:
      condition: on-failure
    update_config:
      parallelism: 1
      delay: 10s
```

**Ce que vous dites :**
> "Pour la production, j'ai adapté le fichier pour Docker Swarm avec un cluster de 1 manager et 2 workers. Les applications web vote et result sont répliquées deux fois pour garantir la haute disponibilité. Si un nœud tombe, Swarm redéploie automatiquement les conteneurs sur les nœuds restants. Les mises à jour se font en rolling update : un conteneur à la fois avec un délai de 10 secondes pour éviter les interruptions de service."

**💡 Différence clé :** Mentionnez que Swarm n'a pas de `build`, d'où la nécessité de construire les images avant.

---

## 🎯 Slide 7 : Démonstration Live (2 min)

**Préparation avant la soutenance :**
```bash
# Terminal 1 - Avoir les commandes prêtes
cd /home/JeremyDebian/projet_ecole/projet_groupe_Supinfo/3DOKR-supinfo
docker compose up -d
```

**Pendant la soutenance :**

**Ce que vous dites :**
> "Je vais maintenant vous faire une démonstration rapide."

**1. Lancement (10 secondes) :**
```bash
docker compose up -d
```
> "Je lance l'application avec une seule commande."

**2. Vérification (10 secondes) :**
```bash
docker compose ps
```
> "Tous les services sont up et healthy."

**3. Application Vote (30 secondes) :**
- Ouvrez http://localhost:5000 dans le navigateur
> "Voici l'interface de vote. Je vote pour 'Cats'."
- Cliquez sur "Cats"

**4. Application Result (30 secondes) :**
- Ouvrez http://localhost:5001 dans le navigateur
> "L'interface de résultats affiche instantanément le vote. La mise à jour est automatique grâce aux WebSockets."

**5. Persistance (20 secondes) :**
```bash
docker compose restart
docker compose ps
```
- Rafraîchissez http://localhost:5001
> "Après redémarrage, les votes sont toujours là grâce au volume PostgreSQL."

**6. Nettoyage (10 secondes) :**
```bash
docker compose down
```

**💡 Conseil :** Ayez les onglets de navigateur pré-ouverts pour gagner du temps.

---

## 🎯 Slide 8 : Résultats & Conformité (30 secondes)

**À l'écran :**
```
Conformité au Cahier des Charges

✅ 3 Dockerfiles optimisés (9/9 points)
✅ Docker Compose complet avec dépendances (9/9 points)
✅ Health checks et persistance (6/6 points)
✅ 2 réseaux isolés (4/4 points)
✅ Docker Swarm documenté et testé (10/10 points)
✅ Documentation complète (2/2 points)

Score attendu: 40/40 points
```

**Ce que vous dites :**
> "Mon projet respecte l'intégralité du cahier des charges. Tous les modules sont conteneurisés avec les bonnes pratiques, Docker Compose gère les dépendances et la persistance, et j'ai préparé un déploiement Swarm production-ready avec haute disponibilité."

---

## 🎯 Slide 9 : Améliorations Futures (30 secondes)

**À l'écran :**
```
Pistes d'Amélioration

🔹 Secrets management (Docker secrets au lieu de variables)
🔹 CI/CD avec GitHub Actions
🔹 Monitoring (Prometheus + Grafana)
🔹 Logs centralisés (ELK Stack)
🔹 Sauvegardes automatisées PostgreSQL
🔹 HTTPS avec certificats SSL
```

**Ce que vous dites :**
> "Pour aller plus loin, on pourrait ajouter la gestion des secrets Docker, mettre en place un pipeline CI/CD, intégrer du monitoring avec Prometheus, et sécuriser les communications avec HTTPS."

**💡 Pourquoi cette slide :** Montre que vous avez une vision complète et que vous savez ce qui manque.

---

## 🎯 Slide 10 : Conclusion (30 secondes)

**À l'écran :**
```
Conclusion

Avant                    →    Après
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
5 scripts bash           →    1 commande
Pas de dépendances       →    Orchestration automatique
Données volatiles        →    Volumes persistants
Pas de scalabilité       →    Cluster Swarm HA
Déploiement manuel       →    Infrastructure as Code

Merci de votre attention.
Questions ?
```

**Ce que vous dites :**
> "En conclusion, j'ai transformé un déploiement manuel complexe en une infrastructure moderne, automatisée et hautement disponible. L'application est maintenant prête pour la production. Je suis prêt à répondre à vos questions."

---

## ❓ Préparation Questions/Réponses (5 points)

### Questions Techniques Probables

**Q: Pourquoi avoir utilisé des images slim/alpine ?**
> "Pour réduire la taille des images, améliorer les temps de téléchargement et réduire la surface d'attaque en termes de sécurité. Par exemple, Python slim fait 150MB contre 1GB pour l'image standard."

**Q: Comment gérez-vous les secrets en production ?**
> "Actuellement, les mots de passe sont en dur dans docker-compose.yml. En production, j'utiliserais Docker secrets avec Swarm, ou des variables d'environnement injectées depuis un vault comme HashiCorp Vault."

**Q: Que se passe-t-il si Redis tombe ?**
> "Redis n'est pas répliqué dans ma configuration actuelle. Les nouveaux votes seraient perdus temporairement. En production, j'utiliserais Redis Sentinel ou Redis Cluster pour la haute disponibilité, ou un volume pour persister les données Redis."

**Q: Pourquoi PostgreSQL et Redis sont sur le manager dans Swarm ?**
> "Pour éviter les problèmes de cohérence des données. Les bases de données stateful nécessitent une gestion particulière. Dans un vrai cluster production, j'utiliserais des solutions dédiées comme un RDS managé ou un stockage distribué avec Ceph."

**Q: Comment testez-vous que les health checks fonctionnent ?**
> "Je peux arrêter manuellement un service et observer que Docker Compose ne démarre pas les services dépendants. Par exemple : `docker compose stop redis && docker compose up worker` → le worker attendra que Redis soit healthy."

**Q: Quelle est la différence entre depends_on classique et avec condition ?**
> "Sans condition, Docker démarre juste les conteneurs dans l'ordre mais ne vérifie pas s'ils sont opérationnels. Avec `condition: service_healthy`, Docker attend que le health check passe avant de démarrer les services dépendants."

**Q: Comment gérez-vous les logs en production ?**
> "Docker stocke les logs par défaut, accessibles via `docker logs`. En production, j'utiliserais un driver de logging comme Fluentd ou un stack ELK pour centraliser et analyser les logs."

**Q: Pourquoi 2 réplicas pour vote et result dans Swarm ?**
> "Pour garantir la haute disponibilité. Si un nœud tombe ou si un conteneur crash, il reste toujours une instance fonctionnelle. Swarm répartit les réplicas sur différents nœuds automatiquement."

**Q: Comment faites-vous une mise à jour sans interruption de service ?**
> "Grâce au rolling update configuré dans Swarm avec `parallelism: 1` et `delay: 10s`. Swarm met à jour un conteneur à la fois, attend qu'il soit healthy, puis passe au suivant."

**Q: Pourquoi séparer les réseaux frontend et backend ?**
> "C'est un principe de sécurité : défense en profondeur. Le worker, Redis et PostgreSQL n'ont pas besoin d'être accessibles depuis l'extérieur, donc ils sont isolés sur le réseau backend. Seules les applications web sont sur frontend."

**Q: Que se passe-t-il si vous faites docker compose down -v ?**
> "Le flag -v supprime les volumes, donc toutes les données PostgreSQL sont perdues. C'est utile pour un reset complet en développement, mais jamais en production."

**Q: Comment sauvegardez-vous PostgreSQL ?**
> "Je créerais un service de backup avec un cron qui exécute `pg_dump` régulièrement et stocke les sauvegardes sur un stockage externe (S3, NFS, etc.)."

---

## 📝 Checklist Avant la Soutenance

### Technique
- [ ] Projet fonctionnel : `docker compose up` marche
- [ ] Images construites et prêtes
- [ ] Navigateur avec onglets pré-ouverts (localhost:5000, localhost:5001)
- [ ] Terminal avec historique de commandes préparé
- [ ] Batterie/alimentation OK pour la démo

### Présentation
- [ ] PowerPoint/PDF prêt (10-12 slides max)
- [ ] Timing répété (15 minutes chrono)
- [ ] Démo testée au moins 2 fois
- [ ] Notes succinctes (pas de lecture intégrale)

### Mental
- [ ] Repos suffisant
- [ ] Arrivée 10 minutes en avance
- [ ] Respiration profonde avant de commencer

---

## 💡 Conseils pour l'Aisance Oratoire (4 points)

1. **Posture** : Debout, dos droit, épaules relâchées
2. **Regard** : Alternez entre l'examinateur et l'écran (70% examinateur / 30% écran)
3. **Voix** : Volume suffisant, débit modéré (pas trop rapide)
4. **Gestuelle** : Mains visibles, gestes naturels pour ponctuer
5. **Pauses** : N'ayez pas peur des silences de 2-3 secondes entre les parties
6. **Enthousiasme** : Montrez que vous êtes fier de votre travail
7. **Mots de transition** : "Passons maintenant à...", "Concernant...", "Pour illustrer..."
8. **Évitez** : "euh", "du coup", "voilà", regarder ses pieds

---

## ⏱️ Simulation de Timing (à faire 2-3 fois avant)

| Partie | Durée | Slide | Timing cumulé |
|--------|-------|-------|---------------|
| Titre | 0:30 | 1 | 0:30 |
| Contexte | 1:30 | 2 | 2:00 |
| Architecture | 2:00 | 3 | 4:00 |
| Dockerfiles | 3:00 | 4 | 7:00 |
| Docker Compose | 3:00 | 5 | 10:00 |
| Docker Swarm | 2:00 | 6 | 12:00 |
| Démo | 2:00 | 7 | 14:00 |
| Résultats | 0:30 | 8 | 14:30 |
| Améliorations | 0:20 | 9 | 14:50 |
| Conclusion | 0:10 | 10 | 15:00 |

---

## 🎬 Script d'Introduction Complet

> "Bonjour [Madame/Monsieur],
> 
> Je vais vous présenter aujourd'hui mon projet de conteneurisation d'une application distribuée de vote. Cette application permet à une audience de choisir entre deux options, avec un affichage des résultats en temps réel.
> 
> Le projet initial était déployé via 5 scripts bash à exécuter manuellement. Mon objectif était de moderniser ce déploiement en utilisant Docker et Docker Swarm pour garantir la fiabilité, la scalabilité et la simplicité d'utilisation.
> 
> Ma présentation durera 15 minutes et sera structurée en 4 parties : l'architecture de l'application, les choix techniques pour les Dockerfiles, l'orchestration avec Docker Compose, et enfin le déploiement en cluster Swarm. Je conclurai par une démonstration live de l'application.
> 
> Commençons par le contexte du projet."

---

## 🎓 Bon à Savoir

- **Si la démo plante** : Restez calme, expliquez ce que vous auriez dû voir, montrez les logs pour diagnostiquer
- **Si on vous interrompt** : Répondez brièvement et proposez de revenir dessus en fin de présentation
- **Si vous ne savez pas** : "Je ne maîtrise pas ce point précisément, mais je pense que..." ou "C'est une bonne question, je vais me renseigner"
- **Si vous êtes en avance** : Ne précipitez pas, faites une pause, demandez si des questions
- **Si vous êtes en retard** : Accélérez légèrement sans bâcler, sautez les détails moins importants

**Dernier conseil : Croyez en votre travail. Vous avez fait un bon projet, montrez-le avec confiance ! 💪**

Bonne soutenance ! 🚀
