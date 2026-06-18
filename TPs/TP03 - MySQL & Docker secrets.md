# TP03 — MySQL & Docker secrets (compose)

> **Niveau :** Avancé
> **Durée estimée :** 40 min
> **Type :** Guidé, étape par étape (pas un exercice).
> **Objectifs :**
> - Comprendre pourquoi mettre un mot de passe en variable d'environnement est risqué.
> - Utiliser les **secrets** de `docker-compose` pour injecter les identifiants MySQL.
> - Vérifier que le secret n'apparaît pas dans la configuration du conteneur.

---

## 1. Pourquoi ne pas utiliser de variables d'environnement ?

Mettre un mot de passe ainsi est une **mauvaise pratique** :

```yaml
environment:
  MYSQL_ROOT_PASSWORD: "SuperP@ssword"
```

Parce que cette valeur est :

- visible en clair dans le `docker-compose.yml` (souvent versionné dans Git) ;
- exposée par `docker inspect` et `docker compose config` ;
- héritée par **tous les processus** du conteneur (lisible dans `/proc/1/environ`) ;
- potentiellement loggée par des outils de supervision.

La solution propre : les **secrets**. Le secret est monté en **fichier** dans le
conteneur (sous `/run/secrets/<nom>`), et n'est jamais exposé comme variable
d'environnement.

> Les images officielles MySQL/MariaDB acceptent un suffixe `_FILE` sur leurs
> variables (ex. `MYSQL_ROOT_PASSWORD_FILE`) : on leur donne le **chemin du
> fichier** secret au lieu de la valeur en clair.

---

## 2. Arborescence du TP

Créez l'arborescence suivante (par exemple dans `TPs/tp03/`) :

```
tp03/
├── docker-compose.yml
└── secrets/
    ├── mysql_root_password.txt
    └── mysql_password.txt
```

---

## 3. Étape par étape

### Étape 1 — Créer les fichiers de secrets

Créez le dossier et les fichiers contenant **uniquement** le mot de passe (sans
retour à la ligne final si possible) :

```bash
cd TPs/tp03
mkdir -p secrets

printf 'R00t-S3cr3t!' > secrets/mysql_root_password.txt
printf 'App-S3cr3t!'  > secrets/mysql_password.txt
```

Restreignez les permissions de ces fichiers :

```bash
chmod 600 secrets/*.txt
```

> ⚠️ Ces fichiers ne doivent **jamais** être versionnés. Ajoutez `secrets/` à
> votre `.gitignore`.

```bash
echo "secrets/" >> .gitignore
```

### Étape 2 — Écrire le `docker-compose.yml`

```yaml
services:

  db:
    image: reg.ntl.nc/proxy/library/mysql:8.0
    environment:
      # On pointe vers les FICHIERS secrets, pas vers les valeurs.
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_password
    secrets:
      - mysql_root_password
      - mysql_password
    volumes:
      - db_data:/var/lib/mysql

# Déclaration des secrets : ici depuis des fichiers locaux.
secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_password:
    file: ./secrets/mysql_password.txt

volumes:
  db_data:
```

Points clés :

- La section **`secrets:` à la racine** déclare les sources (ici, des fichiers).
- La section **`secrets:` dans le service** liste ceux qui sont montés dans ce
  conteneur. Ils apparaissent dans `/run/secrets/<nom>`.
- Les variables `*_FILE` indiquent à l'image MySQL où lire la valeur.

### Étape 3 — Démarrer la base

```bash
docker compose up -d
docker compose logs -f db
```

Attendez le message indiquant que MySQL est prêt :

```
[Server] /usr/sbin/mysqld: ready for connections.
```

### Étape 4 — Vérifier que le secret est bien monté en fichier

```bash
# Le secret est présent en tant que fichier dans le conteneur
docker compose exec db cat /run/secrets/mysql_root_password
# R00t-S3cr3t!
```

### Étape 5 — Vérifier que le mot de passe N'EST PAS en variable d'env

```bash
# Aucune valeur en clair dans l'environnement du conteneur :
docker compose exec db env | grep -i mysql
# On ne voit que ...PASSWORD_FILE=/run/secrets/...  jamais le mot de passe lui-même
```

Comparez avec ce que verrait quelqu'un qui inspecte le conteneur :

```bash
docker inspect $(docker compose ps -q db) | grep -i password
# Seuls les CHEMINS apparaissent, pas les secrets.
```

### Étape 6 — Tester la connexion avec le mot de passe applicatif

```bash
docker compose exec db \
  mysql -u appuser -p"$(cat secrets/mysql_password.txt)" \
  -e "SHOW DATABASES;" appdb
```

Vous devez voir la base `appdb` listée. La connexion fonctionne alors qu'aucun
mot de passe n'a été écrit en clair dans le `docker-compose.yml`.

---

## 4. Nettoyage

```bash
docker compose down -v        # -v supprime aussi le volume db_data
```

---

## 5. À retenir

1. Un secret monté en **fichier** (`/run/secrets/…`) n'est pas exposé via
   `env` / `docker inspect`, contrairement à une variable d'environnement.
2. Les images MySQL/MariaDB acceptent les variables `*_FILE` pour lire les
   identifiants depuis un fichier.
3. Les fichiers de secrets ne se versionnent **jamais** (`.gitignore`), et
   doivent avoir des permissions restreintes (`chmod 600`).
