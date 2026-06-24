# 🐳 Docker Cheatsheet

Aide-mémoire complet des commandes Docker, Docker Compose et Docker Swarm.

---

## Table des matières

- [Informations & Système](#informations--système)
- [Images](#images)
- [Build](#build)
- [Conteneurs (Run)](#conteneurs-run)
- [Gestion des conteneurs](#gestion-des-conteneurs)
- [Logs & Exécution](#logs--exécution)
- [Volumes](#volumes)
- [Networks](#networks)
- [Registry & Hub](#registry--hub)
- [Nettoyage (Prune)](#nettoyage-prune)
- [Docker Compose](#docker-compose)
- [Docker Swarm](#docker-swarm)

---

## Informations & Système

```bash
docker version                  # Version client + serveur
docker info                     # Infos système (conteneurs, images, driver...)
docker --help                   # Aide générale
docker <commande> --help        # Aide d'une commande

docker system df                # Espace disque utilisé par Docker
docker system info              # Infos détaillées du système
docker system events            # Flux d'événements en temps réel
docker stats                    # Stats live (CPU, RAM, I/O) des conteneurs
docker top <conteneur>          # Processus en cours dans un conteneur
```

---

## Images

```bash
docker images                   # Lister les images locales
docker image ls                 # Idem (syntaxe moderne)
docker image ls -a              # Toutes les images (y compris intermédiaires)
docker image ls -q              # Uniquement les IDs

docker pull <image>             # Télécharger une image
docker pull nginx:1.25          # Avec un tag spécifique

docker image inspect <image>    # Détails complets (JSON)
docker history <image>          # Historique des layers d'une image

docker tag <image> <repo>:<tag> # Renommer / taguer une image
docker rmi <image>              # Supprimer une image
docker image rm <image>         # Idem
docker rmi -f <image>           # Forcer la suppression

docker save -o img.tar <image>  # Exporter une image vers un fichier tar
docker load -i img.tar          # Importer une image depuis un tar
```

---

## Build

```bash
docker build -t monapp:1.0 .                 # Build depuis le Dockerfile courant
docker build -t monapp:1.0 -f Dockerfile.prod .   # Dockerfile spécifique
docker build --no-cache -t monapp:1.0 .      # Sans cache
docker build --build-arg ENV=prod -t app .   # Passer un argument de build
docker build --target builder -t app .       # Cibler un stage (multi-stage)
docker build --platform linux/amd64 -t app . # Build multi-architecture

# BuildKit / buildx (multi-plateforme)
docker buildx create --use                   # Créer un builder
docker buildx build --platform linux/amd64,linux/arm64 -t app --push .
docker buildx ls                             # Lister les builders
```

### Exemple de Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Conteneurs (Run)

```bash
docker run <image>                      # Lancer un conteneur
docker run -d <image>                   # En arrière-plan (détaché)
docker run -it <image> bash             # Interactif + terminal
docker run --rm <image>                 # Supprimer à l'arrêt
docker run --name web nginx             # Nommer le conteneur

# Ports
docker run -p 8080:80 nginx             # Mapper port hôte:conteneur
docker run -P nginx                     # Mapper tous les ports exposés (aléatoire)

# Variables d'environnement
docker run -e ENV=prod nginx            # Variable d'env
docker run --env-file .env nginx        # Depuis un fichier

# Volumes
docker run -v /data:/app/data nginx     # Bind mount
docker run -v monvol:/app/data nginx    # Volume nommé
docker run -v $(pwd):/app nginx         # Répertoire courant

# Réseau
docker run --network monreseau nginx    # Réseau spécifique
docker run --hostname web nginx         # Nom d'hôte

# Ressources
docker run --memory=512m nginx          # Limite RAM
docker run --cpus=1.5 nginx             # Limite CPU
docker run --restart=always nginx       # Politique de redémarrage

# Combinaison classique
docker run -d --name web -p 8080:80 --restart unless-stopped nginx
```

---

## Gestion des conteneurs

```bash
docker ps                       # Conteneurs en cours
docker ps -a                    # Tous (y compris arrêtés)
docker ps -q                    # Uniquement les IDs
docker ps -aq                   # Tous les IDs

docker start <conteneur>        # Démarrer
docker stop <conteneur>         # Arrêter (gracieux)
docker restart <conteneur>      # Redémarrer
docker pause <conteneur>        # Suspendre
docker unpause <conteneur>      # Reprendre
docker kill <conteneur>         # Tuer (SIGKILL)

docker rm <conteneur>           # Supprimer
docker rm -f <conteneur>        # Forcer la suppression
docker rm $(docker ps -aq)      # Supprimer tous les conteneurs arrêtés

docker rename <ancien> <nouveau>   # Renommer
docker inspect <conteneur>      # Détails (JSON)
docker port <conteneur>         # Mappings de ports
docker diff <conteneur>         # Changements du système de fichiers
```

---

## Logs & Exécution

```bash
docker logs <conteneur>             # Afficher les logs
docker logs -f <conteneur>          # Suivre en temps réel
docker logs --tail 100 <conteneur>  # 100 dernières lignes
docker logs -t <conteneur>          # Avec timestamps
docker logs --since 10m <conteneur> # Depuis 10 minutes

docker exec -it <conteneur> bash    # Ouvrir un shell
docker exec -it <conteneur> sh      # Shell (Alpine)
docker exec <conteneur> ls /app     # Exécuter une commande
docker exec -u root <conteneur> sh  # En tant qu'utilisateur root

docker attach <conteneur>           # S'attacher au processus principal
docker cp <conteneur>:/chemin ./local   # Copier depuis le conteneur
docker cp ./local <conteneur>:/chemin   # Copier vers le conteneur

docker commit <conteneur> nouvelleimage # Créer une image depuis un conteneur
docker export <conteneur> -o cont.tar   # Exporter le FS d'un conteneur
```

---

## Volumes

```bash
docker volume create monvol         # Créer un volume
docker volume ls                    # Lister les volumes
docker volume inspect monvol        # Détails d'un volume
docker volume rm monvol             # Supprimer un volume
docker volume prune                 # Supprimer les volumes inutilisés
docker volume prune -a              # Supprimer tous les volumes non utilisés

# Utilisation
docker run -v monvol:/data nginx                    # Volume nommé
docker run -v /chemin/hote:/data nginx              # Bind mount
docker run -v /data nginx                           # Volume anonyme
docker run --mount source=monvol,target=/data nginx # Syntaxe --mount
docker run -v monvol:/data:ro nginx                 # Lecture seule (ro)
```

---

## Networks

```bash
docker network ls                       # Lister les réseaux
docker network create monreseau         # Créer un réseau (bridge par défaut)
docker network create -d bridge net1    # Driver bridge
docker network create -d overlay net1   # Driver overlay (Swarm)
docker network create --subnet 172.20.0.0/16 net1   # Sous-réseau custom

docker network inspect monreseau        # Détails du réseau
docker network rm monreseau             # Supprimer un réseau
docker network prune                    # Supprimer les réseaux inutilisés

docker network connect monreseau web    # Connecter un conteneur
docker network disconnect monreseau web # Déconnecter un conteneur

# Lancer un conteneur sur un réseau
docker run --network monreseau --name web nginx
docker run --network host nginx         # Réseau de l'hôte
docker run --network none nginx         # Aucun réseau
```

---

## Registry & Hub

```bash
docker login                            # Se connecter à Docker Hub
docker login registry.example.com       # Registry privé
docker logout                           # Se déconnecter

docker push <repo>:<tag>                # Pousser une image
docker pull <repo>:<tag>                # Tirer une image
docker search nginx                     # Rechercher sur Docker Hub
```

---

## Nettoyage (Prune)

```bash
docker container prune          # Conteneurs arrêtés
docker image prune              # Images dangling (sans tag)
docker image prune -a           # Toutes les images inutilisées
docker volume prune             # Volumes inutilisés
docker network prune            # Réseaux inutilisés

docker system prune             # Tout ce qui est inutilisé
docker system prune -a          # + images non utilisées
docker system prune -a --volumes   # + volumes (nettoyage total)
```

---

# 🐙 Docker Compose

> Selon la version : `docker-compose` (v1) ou `docker compose` (v2, intégré).

## Cycle de vie

```bash
docker compose up                   # Démarrer les services
docker compose up -d                # En arrière-plan
docker compose up --build           # Rebuild avant de démarrer
docker compose up --force-recreate  # Recréer les conteneurs
docker compose up service1 service2 # Services spécifiques

docker compose down                 # Arrêter et supprimer
docker compose down -v              # + supprimer les volumes
docker compose down --rmi all       # + supprimer les images

docker compose start                # Démarrer les services existants
docker compose stop                 # Arrêter sans supprimer
docker compose restart              # Redémarrer
docker compose pause / unpause      # Suspendre / reprendre
```

## Build & Images

```bash
docker compose build                # Construire les images
docker compose build --no-cache     # Sans cache
docker compose pull                 # Tirer les images
docker compose push                 # Pousser les images
```

## Inspection & Debug

```bash
docker compose ps                   # Lister les conteneurs
docker compose ps -a                # Tous
docker compose logs                 # Logs de tous les services
docker compose logs -f              # Suivre
docker compose logs -f service1     # Service spécifique

docker compose top                  # Processus en cours
docker compose config               # Valider et afficher la config
docker compose images               # Images utilisées
docker compose events               # Flux d'événements
```

## Exécution

```bash
docker compose exec service1 bash   # Shell dans un service
docker compose run service1 cmd     # Lancer une commande ponctuelle
docker compose run --rm service1 sh # Avec suppression auto

docker compose scale service1=3     # Mettre à l'échelle (legacy)
docker compose up -d --scale web=3  # Mettre à l'échelle (v2)
```

## Fichiers & Profils

```bash
docker compose -f docker-compose.prod.yml up     # Fichier spécifique
docker compose -f base.yml -f override.yml up    # Plusieurs fichiers
docker compose --profile debug up                # Activer un profil
docker compose -p monprojet up                   # Nom de projet custom
```

### Exemple de docker-compose.yml

```yaml
services:
  web:
    build: .
    image: monapp:1.0
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    volumes:
      - ./src:/app/src
    networks:
      - frontend
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - frontend

volumes:
  db-data:

networks:
  frontend:
```

---

# 🐝 Docker Swarm

## Initialisation du cluster

```bash
docker swarm init                                   # Initialiser le swarm
docker swarm init --advertise-addr <IP>             # Avec IP spécifique
docker swarm join-token worker                      # Token pour les workers
docker swarm join-token manager                     # Token pour les managers
docker swarm join --token <token> <IP>:2377         # Rejoindre un swarm
docker swarm leave                                  # Quitter le swarm
docker swarm leave --force                          # Forcer (manager)
docker swarm update --autolock=true                 # Verrouillage auto
docker swarm unlock                                 # Déverrouiller
```

## Gestion des nœuds

```bash
docker node ls                          # Lister les nœuds
docker node inspect <node>              # Détails d'un nœud
docker node ps <node>                   # Tâches d'un nœud
docker node update --availability drain <node>   # Vider un nœud
docker node update --availability active <node>  # Réactiver un nœud
docker node update --role manager <node>         # Promouvoir manager
docker node promote <node>              # Promouvoir
docker node demote <node>               # Rétrograder
docker node rm <node>                   # Supprimer un nœud
docker node update --label-add zone=eu <node>    # Ajouter un label
```

## Services

```bash
docker service create --name web nginx              # Créer un service
docker service create --name web --replicas 3 -p 80:80 nginx
docker service create --name web --mode global nginx   # 1 par nœud

docker service ls                       # Lister les services
docker service ps web                   # Tâches d'un service
docker service inspect web              # Détails (JSON)
docker service logs web                 # Logs
docker service logs -f web              # Suivre les logs

docker service scale web=5              # Mettre à l'échelle
docker service scale web=5 api=2        # Plusieurs services

docker service update --image nginx:1.25 web        # Mettre à jour l'image
docker service update --replicas 4 web              # Changer le nb de réplicas
docker service update --env-add KEY=val web         # Ajouter une variable
docker service update --publish-add 8080:80 web     # Ajouter un port

docker service rollback web             # Revenir à la version précédente
docker service rm web                   # Supprimer un service
```

## Stacks (déploiement multi-services)

```bash
docker stack deploy -c docker-compose.yml monstack  # Déployer une stack
docker stack ls                         # Lister les stacks
docker stack services monstack          # Services d'une stack
docker stack ps monstack                # Tâches d'une stack
docker stack rm monstack                # Supprimer une stack
```

## Secrets & Configs

```bash
# Secrets
echo "monsecret" | docker secret create db_pass -   # Créer un secret
docker secret ls                        # Lister
docker secret inspect db_pass           # Détails
docker secret rm db_pass                # Supprimer
docker service create --secret db_pass nginx        # Utiliser

# Configs
docker config create monconf ./fichier.conf         # Créer une config
docker config ls                        # Lister
docker config inspect monconf           # Détails
docker config rm monconf                # Supprimer
```

### Exemple de stack (docker-compose.yml pour Swarm)

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
```

---

## 💡 Astuces

```bash
# Supprimer tous les conteneurs
docker rm -f $(docker ps -aq)

# Supprimer toutes les images
docker rmi -f $(docker images -q)

# Inspecter une valeur précise avec un format
docker inspect -f '{{.NetworkSettings.IPAddress}}' <conteneur>

# Voir l'IP d'un conteneur
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <conteneur>

# Lister les conteneurs avec un format custom
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```
