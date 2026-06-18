# TP01 — Bad Dockerfile (débogage Flask)

> **Niveau :** Avancé
> **Durée estimée :** 45 min
> **Objectifs :**
> - Réparer un Dockerfile pour que l'application démarre.
> - Optimiser l'ordre des instructions du Dockerfile.

---

## 1. Contexte

On vous livre une application Flask et son `Dockerfile`. Le build réussit, mais
le conteneur **plante au démarrage**.

À vous de le réparer puis de l'optimiser.

---

## 2. Fichiers fournis

Créez l'arborescence suivante (par exemple dans `TPs/tp01/`) :

```
tp01/
├── Dockerfile
├── requirements.txt
└── src/
    └── app.py
```

### `src/app.py`

```python
from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello():
    return "TP01 — l'application Flask tourne !\n"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### `requirements.txt`

```
flask
```

### `Dockerfile`

```dockerfile
FROM reg.ntl.nc/proxy/library/python:3.12-slim

COPY src/ .

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

WORKDIR /app

CMD ["python", "app.py"]
```

---

## 3. Consignes

```bash
cd TPs/tp01

docker build -t tp01 .
docker run --rm -p 5000:5000 tp01
```

1. **Réparez le `Dockerfile`** pour que l'application démarre.

   Critère de réussite :

   ```bash
   curl http://localhost:5000
   # TP01 — l'application Flask tourne !
   ```

2. **Optimisez l'ordre des instructions** du `Dockerfile` pour que la
   modification du code applicatif ne provoque plus la réinstallation des
   dépendances.

   Critère de réussite : après une modification de `src/app.py`, le `docker build`
   affiche l'étape d'installation des dépendances en `CACHED`.
