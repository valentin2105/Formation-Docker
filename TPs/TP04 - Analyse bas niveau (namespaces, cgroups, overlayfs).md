# TP04 — Analyse bas niveau : namespaces, cgroups, overlayfs

> **Niveau :** Avancé (public sysadmin / Linux)
> **Durée estimée :** 1 h
> **Type :** Guidé, étape par étape (pas un exercice).
> **Pré-requis :** la formation tourne **directement sur Ubuntu 26.04** (cgroups v2
> par défaut). Toutes les commandes s'exécutent sur cet hôte.
> **Objectifs :**
> - Comprendre qu'un conteneur n'est **pas une machine virtuelle**, mais un
>   processus Linux ordinaire isolé par le noyau.
> - Observer les **namespaces**, les **cgroups** et le stockage **overlay2**.
> - Comprendre pourquoi les couches d'image sont en **lecture seule**.

---

## 1. Idée directrice

Un conteneur, c'est trois mécanismes du noyau Linux assemblés :

| Mécanisme | Rôle | Ce qu'il isole / limite |
|-----------|------|--------------------------|
| **namespaces** | isolation | ce que le processus *voit* (PID, réseau, montages, hostname…) |
| **cgroups** | quotas | ce que le processus *consomme* (CPU, RAM, I/O…) |
| **overlayfs** | stockage | un système de fichiers en couches (image en lecture seule + couche d'écriture) |

Docker orchestre tout ça, mais ce sont des outils Linux standards. On va les
observer « à la main ».

---

## 2. Mise en place

On lance un conteneur de test avec une limite mémoire (utile pour les cgroups),
puis on le laisse tourner :

```bash
docker run -d --name lab \
  --memory=256m --cpus=0.5 \
  reg.ntl.nc/proxy/library/python:3.12-slim \
  python -m http.server 8000
```

Récupérez le **PID du processus sur l'hôte** :

```bash
PID=$(docker inspect -f '{{.State.Pid}}' lab)
echo "PID hôte du conteneur = $PID"
```

> Retenez bien : ce PID est celui que voit **l'hôte**. Dans le conteneur, ce même
> processus sera vu comme **PID 1**. C'est tout l'enjeu du namespace PID.

---

## 3. Étape par étape

### Étape 1 — Comparer les processus hôte / conteneur

Vue depuis l'**hôte** (on voit tous les processus, dont celui du conteneur) :

```bash
ps aux | grep "http.server"
# le processus existe avec un PID "normal" de l'hôte (ex: 24531)
```

Vue depuis le **conteneur**. L'image `slim` ne contient pas `ps`, on lit donc
directement `/proc` (toujours présent) :

```bash
# Liste des PID visibles DANS le conteneur : seulement les siens
docker exec lab sh -c 'ls -d /proc/[0-9]*'
# /proc/1  (et éventuellement quelques sous-processus)

# Le serveur http est bien le PID 1 du conteneur
docker exec lab sh -c 'cat /proc/1/cmdline | tr "\0" " "; echo'
# python -m http.server 8000
```

➡️ Même processus, deux numéros différents (PID `$PID` côté hôte, PID 1 côté
conteneur), et le conteneur ne voit pas les processus de l'hôte. C'est le
**namespace PID** en action.

### Étape 2 — Inspecter les namespaces

Lister les namespaces du système et leurs processus :

```bash
sudo lsns
```

Cibler ceux du conteneur via son PID hôte :

```bash
sudo lsns -p $PID
# colonnes : NS  TYPE  NPROCS  PID  USER  COMMAND
# on voit des namespaces de type mnt, net, pid, uts, ipc, cgroup...
```

Comparer avec les namespaces du processus `init` de l'hôte (PID 1) :

```bash
sudo ls -l /proc/1/ns/      # namespaces de l'hôte
sudo ls -l /proc/$PID/ns/   # namespaces du conteneur
```

➡️ Les identifiants (les numéros entre crochets) **diffèrent** : le conteneur
vit dans d'autres namespaces que l'hôte. Là où ils sont **identiques**, c'est
qu'il **partage** ce namespace avec l'hôte (selon la configuration).

### Étape 3 — Entrer dans les namespaces du conteneur (nsenter)

`nsenter` exécute une commande **dans les namespaces** d'un processus existant —
sans passer par `docker exec`. La commande lancée est le **binaire de l'hôte**,
mais elle « voit » l'environnement du conteneur. C'est idéal pour le réseau :

```bash
# Le réseau de l'hôte (référence)
ip -brief addr

# Le réseau VU depuis le conteneur (binaire ip de l'hôte, namespace net du conteneur)
sudo nsenter -t $PID -n ip -brief addr
# on voit eth0 avec l'IP du conteneur, pas les interfaces de l'hôte
```

On peut aussi entrer dans le namespace UTS (hostname) :

```bash
sudo nsenter -t $PID -u hostname
# affiche l'ID du conteneur comme hostname, différent de celui de l'hôte
```

➡️ `nsenter` prouve qu'un conteneur n'est qu'un ensemble de namespaces : on peut
y « entrer » avec un outil système standard.

### Étape 4 — Voir les cgroups (limites de ressources)

Les cgroups appliquent les quotas (`--memory`, `--cpus`). Ubuntu 26.04 utilise
**cgroups v2** :

```bash
# Le chemin cgroup du processus
cat /proc/$PID/cgroup
# ex: 0::/system.slice/docker-<ID>.scope
```

Lire les limites effectives :

```bash
CG=/sys/fs/cgroup$(cat /proc/$PID/cgroup | cut -d: -f3)

cat $CG/memory.max        # ~268435456 (256 Mo) -> notre --memory=256m
cat $CG/memory.current    # mémoire réellement consommée
cat $CG/cpu.max           # quota CPU -> reflète --cpus=0.5
```

Vue synthétique via Docker :

```bash
docker stats --no-stream lab
# MEM USAGE / LIMIT montre la limite de 256 Mo appliquée par le cgroup
```

➡️ Les limites ne sont pas « simulées » par Docker : elles sont **imposées par le
noyau** via les fichiers du cgroup.

### Étape 5 — Explorer le stockage overlay2

Regardez comment le système de fichiers du conteneur est monté :

```bash
mount | grep overlay
# overlay on /var/lib/docker/overlay2/<id>/merged type overlay
#   (lowerdir=...:...:...,upperdir=...,workdir=...)
```

Détail des répertoires (via Docker) :

```bash
docker inspect -f '{{ json .GraphDriver.Data }}' lab | tr ',' '\n'
```

Vous y trouverez trois notions clés d'overlayfs :

| Répertoire | Rôle |
|------------|------|
| **lowerdir** | les **couches de l'image**, empilées, en **lecture seule** |
| **upperdir** | la **couche d'écriture** du conteneur (les modifications) |
| **merged**   | la **vue fusionnée** : ce que « voit » le conteneur (lower + upper) |

### Étape 6 — Comprendre pourquoi les couches sont en lecture seule (copy-on-write)

Faites une modification **dans le conteneur** puis observez où elle atterrit :

```bash
docker exec lab sh -c "echo 'coucou' > /tmp/test.txt"

# Le fichier n'existe PAS dans les couches d'image (lowerdir),
# il apparaît dans la couche d'écriture (upperdir) :
UPPER=$(docker inspect -f '{{ .GraphDriver.Data.UpperDir }}' lab)
sudo find $UPPER -name test.txt
# .../diff/tmp/test.txt
```

Modifier un fichier qui **existe déjà dans l'image** déclenche un
**copy-on-write** : overlay2 copie le fichier depuis la couche image (lecture
seule) vers la couche d'écriture, puis applique la modification dessus.

➡️ Les couches d'image restent **immuables et partagées** entre tous les
conteneurs issus de la même image. C'est ce qui rend les images :

- **légères** : on ne duplique pas les couches communes ;
- **reproductibles** : une couche ne change jamais (son hash l'identifie) ;
- **rapides à démarrer** : seule la fine couche d'écriture est créée au `run`.

Preuve du partage : lancez un 2ᵉ conteneur de la même image et constatez qu'ils
partagent les mêmes `lowerdir` mais ont chacun leur `upperdir`.

```bash
docker run -d --name lab2 reg.ntl.nc/proxy/library/python:3.12-slim sleep 600
docker inspect -f '{{ .GraphDriver.Data.LowerDir }}' lab  | tr ':' '\n' | head
docker inspect -f '{{ .GraphDriver.Data.LowerDir }}' lab2 | tr ':' '\n' | head
# les couches de base communes sont identiques
```

---

## 4. Nettoyage

```bash
docker rm -f lab lab2
```

---

## 5. À retenir

1. Un conteneur = **un processus de l'hôte**, isolé par des **namespaces** et
   limité par des **cgroups**. Pas de noyau dédié, pas de VM.
2. `lsns`, `nsenter`, `/proc/<pid>/ns`, `/sys/fs/cgroup` permettent de tout
   observer avec des outils **Linux standards** : Docker n'invente rien, il
   assemble.
3. overlay2 empile des **couches en lecture seule** (image) sous une **couche
   d'écriture** (conteneur), avec **copy-on-write** : d'où des images légères,
   partagées et immuables.
