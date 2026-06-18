# TP02 — Multi-stage build (Go)

> **Niveau :** Avancé
> **Durée estimée :** 45 min
> **Objectifs :**
> - Comprendre le poids d'une image construite « en une seule étape ».
> - Transformer un Dockerfile mono-étape en **multi-stage build**.
> - Réduire drastiquement la taille de l'image finale.

---

## 1. Contexte

On vous livre un petit serveur HTTP en Go et un `Dockerfile` qui le construit
**en une seule étape** : l'image finale embarque tout le toolchain Go (compilateur,
modules, cache…) alors que le binaire compilé se suffit à lui-même.

À vous de la transformer en **multi-stage build** pour ne livrer que le binaire.

---

## 2. Fichiers fournis

Créez l'arborescence suivante (par exemple dans `TPs/tp02/`) :

```
tp02/
├── Dockerfile
├── go.mod
└── main.go
```

### `main.go`

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "TP02 — Hello World depuis Go !")
	})

	fmt.Println("Serveur démarré sur :8080")
	http.ListenAndServe(":8080", nil)
}
```

### `go.mod`

```
module tp02

go 1.22
```

### `Dockerfile` (mono-étape)

```dockerfile
FROM reg.ntl.nc/proxy/library/golang:1.22

WORKDIR /app

COPY go.mod .
COPY main.go .

RUN go build -o server main.go

EXPOSE 8080

CMD ["./server"]
```

---

## 3. Consignes

```bash
cd TPs/tp02

docker build -t tp02:mono .
docker run --rm -p 8080:8080 tp02:mono
```

Vérifiez que l'application répond :

```bash
curl http://localhost:8080
# TP02 — Hello World depuis Go !
```

Regardez la taille de l'image obtenue :

```bash
docker images tp02:mono
```

1. **Transformez le `Dockerfile` en multi-stage build** : une étape pour compiler
   le binaire, une étape finale minimale qui ne contient **que** le binaire.

   Critère de réussite : l'application répond toujours sur `http://localhost:8080`
   et l'image finale est **drastiquement plus petite** que `tp02:mono`.

   ```bash
   docker build -t tp02:multi .
   docker run --rm -p 8080:8080 tp02:multi
   curl http://localhost:8080

   docker images | grep tp02
   # comparez la taille de tp02:mono et tp02:multi
   ```
