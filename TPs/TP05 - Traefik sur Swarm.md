# TP05 — Traefik sur Docker Swarm (reverse proxy dynamique)

> **Niveau :** Avancé
> **Durée estimée :** 1 h
> **Type :** Guidé, étape par étape (pas un exercice).
> **Pré-requis :**
> - Un cluster **Docker Swarm** fonctionnel (au moins 1 manager + 1 worker).
>   Un seul nœud suffit pour le TP si l'on relâche les contraintes de placement.
> - Docker en version récente (Traefik **v3.6** est indispensable, voir §6).
> - `jq` installé pour interroger l'API Traefik.
> **Objectifs :**
> - Comprendre le rôle d'un **reverse proxy** dans un cluster Swarm.
> - Déployer **Traefik v3.6** comme point d'entrée unique du cluster.
> - Publier un service applicatif (`hello`) **sans toucher à Traefik**, uniquement
>   via des **labels** sur le service.
> - Comprendre le **routage dynamique** (découverte automatique des services).

---

## 1. Idée directrice

Sans reverse proxy, chaque service exposé doit publier son propre port sur l'hôte
(`-p 8081:80`, `-p 8082:80`…). Cela devient vite ingérable : conflits de ports,
pas de routage par nom de domaine, pas de TLS centralisé.

**Traefik** résout ça. C'est un reverse proxy qui :

- écoute sur **un seul port public** (ici `:80`) ;
- **découvre tout seul** les services à exposer en lisant le **socket Docker** ;
- route chaque requête vers le bon service selon des **règles** (`Host`, `Path`…) ;
- se reconfigure **à chaud** quand un service apparaît, scale ou disparaît.

La configuration d'un service exposé ne vit donc **pas** dans Traefik, mais sur le
**service lui-même**, sous forme de **labels**. C'est le principe du *provider*
Docker/Swarm.

| Concept Traefik | Rôle |
|-----------------|------|
| **entrypoint**  | un port d'écoute (ex. `web` = `:80`) |
| **router**      | une règle (`Host(...)`) qui dirige le trafic vers un service |
| **service**     | la cible interne (le conteneur applicatif et son port) |
| **provider**    | la source de configuration (ici, le **socket Docker en mode Swarm**) |

---

## 2. Schéma des flux

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                   Nœud MANAGER                          │
   navigateur       │                                                         │
 (votre poste)      │   ┌──────────────────────────────────────────────┐     │
       │            │   │                 TRAEFIK v3.6                  │     │
       │            │   │                                              │     │
  http://hello      │   │  entrypoint web  :80   ───► router "hello"   │     │
   .localhost   ────┼──►│  dashboard       :8080      rule=Host(...)   │     │
       │   :80      │   │                                  │           │     │
       │            │   │   lit le socket Docker ◄─────────┤           │     │
       │            │   │   /var/run/docker.sock (ro)      │           │     │
       │            │   └──────────────────────────────────┼───────────┘     │
       │            │                                       │                 │
       │            │            réseau overlay  "traefik-public"             │
       │            │                                       │                 │
       └────────────┘             ┌─────────────────────────┼───────┐         │
                                  ▼                         ▼       │         │
                    ┌──────────────────────┐  ┌──────────────────────┐        │
                    │   Nœud WORKER #1     │  │   Nœud WORKER #2     │        │
                    │  service hello (1/2)  │  │  service hello (2/2) │        │
                    │  nginxdemos/hello :80 │  │  nginxdemos/hello :80│        │
                    └──────────────────────┘  └──────────────────────┘        │
                    └─────────────────────────────────────────────────────────┘

Flux d'une requête :
 1. Le navigateur résout  hello.localhost → 127.0.0.1  (TLD .localhost = loopback).
 2. La requête arrive sur l'entrypoint  web (:80)  de Traefik, sur le manager.
 3. Traefik a découvert le service via les LABELS lus sur le socket Docker.
 4. Le router "hello" matche  Host(`hello.localhost`)  → service hello.
 5. Traefik route via le réseau overlay "traefik-public" vers un des 2 réplicas
    (load-balancing round-robin assuré par Traefik).
```

Trois éléments rendent ce montage possible (à garder en tête) :

1. **Traefik en v3.6** : auto-négociation de l'API Docker côté Swarm (corrige les
   `404` rencontrés sur des versions antérieures).
2. **Labels sous `deploy.labels`** (et non au niveau conteneur) : en Swarm, c'est
   le service qui porte la configuration, pas la tâche.
3. **Réseau `traefik-public` externe** partagé : évite le préfixage par nom de
   stack et garantit que Traefik et `hello` sont sur le **même** overlay.

---

## 3. Mise en place du réseau partagé

Traefik et les applications doivent communiquer sur un **même réseau overlay**.
On le crée **une seule fois**, en dehors des stacks, pour qu'il soit partagé :

```bash
docker network create --driver overlay --attachable traefik-public
```

- `--driver overlay` : réseau multi-hôtes, requis en Swarm.
- `--attachable` : pratique pour y attacher manuellement un conteneur de debug.

> Déclaré `external: true` dans les stacks, ce réseau ne sera **pas** préfixé par
> le nom de la stack (sinon on aurait `traefik_traefik-public` et `hello_traefik-public`,
> deux réseaux distincts qui ne se verraient pas).

---

## 4. Déployer Traefik

Créez `traefik-stack.yml` :

```yaml
# traefik-stack.yml
services:
  traefik:
    image: traefik:v3.6
    command:
      # --- Provider Swarm : Traefik lit les services Swarm via le socket Docker
      - "--providers.swarm=true"
      - "--providers.swarm.endpoint=unix:///var/run/docker.sock"
      - "--providers.swarm.exposedByDefault=false"   # rien n'est exposé sans label explicite
      - "--providers.swarm.network=traefik-public"    # réseau par défaut pour joindre les services
      # --- Entrypoint HTTP
      - "--entrypoints.web.address=:80"
      # --- Dashboard (mode non sécurisé : pour le TP uniquement)
      - "--api.dashboard=true"
      - "--api.insecure=true"
    ports:
      - "80:80"      # trafic applicatif
      - "8080:8080"  # dashboard / API Traefik
    volumes:
      # Lecture seule : Traefik n'a besoin que de LIRE l'état du cluster.
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-public
    deploy:
      placement:
        constraints:
          # Le socket Docker du manager expose l'API Swarm : Traefik doit y tourner.
          - node.role == manager

networks:
  traefik-public:
    external: true
```

Points clés :

- `exposedByDefault=false` : **sécurité par défaut**. Un service n'est exposé que
  s'il porte `traefik.enable=true`. Sans ça, tous vos services seraient publiés.
- Le **socket Docker** est monté en **lecture seule** (`:ro`) : Traefik observe le
  cluster, il ne le modifie pas.
- La contrainte `node.role == manager` est **obligatoire** : seul le socket d'un
  manager donne accès à l'API Swarm (la liste des services).
- `--api.insecure=true` expose le dashboard sans authentification sur `:8080`.
  **À ne jamais faire en production** — ici c'est uniquement pour observer.

Déployez :

```bash
docker stack deploy -c traefik-stack.yml traefik
docker service ls | grep traefik     # traefik_traefik   1/1
```

Vérifiez que le dashboard répond :

```bash
curl -s http://localhost:8080/api/overview | jq '.providers'
# doit lister "swarm"
```

Ou ouvrez **http://localhost:8080/dashboard/** dans le navigateur.

---

## 5. Déployer l'application `hello`

Créez `hello-stack.yml` :

```yaml
# hello-stack.yml
services:
  hello:
    image: nginxdemos/hello       # petite image qui affiche infos requête + nom du conteneur
    networks:
      - traefik-public
    deploy:
      replicas: 2
      restart_policy:
        condition: any
      placement:
        constraints:
          - node.role == worker    # cf. note ci-dessous si cluster mono-nœud
      labels:
        # TOUTE la config Traefik tient dans ces labels, portés par le SERVICE.
        - "traefik.enable=true"
        - "traefik.http.routers.hello.rule=Host(`hello.localhost`)"
        - "traefik.http.routers.hello.entrypoints=web"
        - "traefik.http.services.hello.loadbalancer.server.port=80"
        - "traefik.swarm.network=traefik-public"

networks:
  traefik-public:
    external: true
```

Décortiquons les labels :

| Label | Signification |
|-------|---------------|
| `traefik.enable=true` | « Expose-moi » (obligatoire car `exposedByDefault=false`). |
| `routers.hello.rule=Host(...)` | Crée un **router** nommé `hello` qui matche ce domaine. |
| `routers.hello.entrypoints=web` | Ce router écoute sur l'entrypoint `web` (`:80`). |
| `services.hello.loadbalancer.server.port=80` | Port **interne** du conteneur où Traefik envoie le trafic. |
| `traefik.swarm.network=traefik-public` | Réseau que Traefik utilise pour joindre ce service. |

> **⚠️ Labels sous `deploy.labels`**, pas au niveau du conteneur. En Swarm, les
> labels de conteneur ne sont pas vus par le provider `swarm` ; il lit les labels
> du **service** (donc sous `deploy:`). C'est une erreur très fréquente.

> **⚠️ `traefik.swarm.network`** est la forme **à jour en v3.6**. Elle remplace
> l'ancien `traefik.docker.network` des versions précédentes.

Déployez :

```bash
docker stack deploy -c hello-stack.yml hello
docker service ls | grep hello       # hello_hello   2/2
```

> Sur notre cluster **4 nœuds**, la contrainte `node.role == worker` répartit les
> 2 réplicas sur les workers, tandis que Traefik reste sur le manager (cf. §4).
> Séparer le rôle « entrée » (manager) du rôle « charge applicative » (workers)
> est une bonne pratique.

---

## 6. Vérifier le routage

### Étape 1 — Le router est bien découvert

```bash
curl -s http://localhost:8080/api/http/routers | jq '.[].name'
# → "hello@swarm"
```

Le suffixe `@swarm` confirme que la route vient du **provider Swarm** (et pas d'un
fichier statique). Vous pouvez aussi le voir dans le dashboard, onglet *HTTP →
Routers*.

### Étape 2 — Accéder à l'application

Le TLD **`.localhost`** est résolu vers `127.0.0.1` par la quasi-totalité des
systèmes et navigateurs — **aucune entrée `/etc/hosts` n'est nécessaire**.

```bash
curl http://hello.localhost/
```

Ou ouvrez **http://hello.localhost/** dans le navigateur : la page `nginxdemos/hello`
affiche les infos de la requête et le **nom du conteneur** qui a répondu.

### Étape 3 — Observer le load-balancing

Rafraîchissez plusieurs fois (ou en boucle) : le champ *Server name* / *Server
address* alterne entre les **2 réplicas**.

```bash
for i in $(seq 1 6); do
  curl -s http://hello.localhost/ | grep -i 'Server name'
done
# le hostname renvoyé alterne entre les deux conteneurs → round-robin
```

### Étape 4 — Routage dynamique (le cœur du sujet)

Scalez le service **sans toucher à Traefik** :

```bash
docker service scale hello_hello=4
```

Quelques secondes plus tard, Traefik a **automatiquement** intégré les 2 nouveaux
réplicas dans son load-balancer (relancez la boucle de l'étape 3 : 4 hostnames
différents apparaissent). Aucune reconfiguration, aucun redémarrage de Traefik :
c'est la **découverte dynamique** via le socket Docker.

---

## 7. Nettoyage

```bash
docker stack rm hello
docker stack rm traefik
docker network rm traefik-public
```

---

## 8. À retenir

1. **Un seul point d'entrée** (Traefik sur `:80`) suffit à exposer N services :
   plus de gestion manuelle des ports, routage par nom de domaine.
2. La configuration d'exposition vit **sur le service** (`deploy.labels`), pas
   dans Traefik. Traefik la **découvre** en lisant le socket Docker du manager.
3. Le trio gagnant en Swarm : **Traefik v3.6** (auto-négociation API), **labels
   sous `deploy.labels`**, **réseau overlay `external` partagé**. Et
   `traefik.swarm.network` remplace l'ancien `traefik.docker.network`.
4. `exposedByDefault=false` + `traefik.enable=true` = sécurité par défaut.
   Le dashboard `--api.insecure` est un confort de TP, **jamais en production**.
