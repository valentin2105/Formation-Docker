# ☸️ Kubectl Cheatsheet

Aide-mémoire complet des commandes `kubectl` pour Kubernetes.

---

## Table des matières

- [Configuration & Contexte](#configuration--contexte)
- [Informations sur le cluster](#informations-sur-le-cluster)
- [Nodes](#nodes)
- [Pods](#pods)
- [Deployments](#deployments)
- [ReplicaSets & DaemonSets](#replicasets--daemonsets)
- [Services](#services)
- [Ingress](#ingress)
- [Namespaces](#namespaces)
- [ConfigMaps & Secrets](#configmaps--secrets)
- [Volumes (PV / PVC)](#volumes-pv--pvc)
- [Jobs & CronJobs](#jobs--cronjobs)
- [StatefulSets](#statefulsets)
- [Logs & Debug](#logs--debug)
- [Apply / Create / Delete](#apply--create--delete)
- [Scaling & Rollout](#scaling--rollout)
- [Labels & Annotations](#labels--annotations)
- [RBAC](#rbac)
- [Astuces](#astuces)

---

## Configuration & Contexte

```bash
kubectl config view                         # Afficher la config (kubeconfig)
kubectl config get-contexts                 # Lister les contextes
kubectl config current-context              # Contexte actuel
kubectl config use-context <contexte>       # Changer de contexte
kubectl config set-context --current --namespace=<ns>   # Namespace par défaut
kubectl config get-clusters                 # Lister les clusters
kubectl config delete-context <contexte>    # Supprimer un contexte

kubectl version                             # Version client + serveur
kubectl api-resources                       # Lister les ressources disponibles
kubectl api-versions                        # Lister les versions d'API
kubectl explain pod                         # Documentation d'une ressource
kubectl explain pod.spec.containers         # Doc d'un champ précis
```

---

## Informations sur le cluster

```bash
kubectl cluster-info                        # Infos du cluster
kubectl cluster-info dump                   # Dump complet (debug)
kubectl get componentstatuses              # État des composants
kubectl get events                          # Événements du cluster
kubectl get events --sort-by='.lastTimestamp'   # Triés par date
kubectl top node                            # Consommation des nodes
kubectl top pod                             # Consommation des pods
kubectl top pod --containers                # Détail par conteneur
```

---

## Nodes

```bash
kubectl get nodes                           # Lister les nodes
kubectl get nodes -o wide                   # Avec IP, OS, version
kubectl describe node <node>                # Détails d'un node
kubectl cordon <node>                       # Marquer non-schedulable
kubectl uncordon <node>                     # Réautoriser le scheduling
kubectl drain <node>                        # Vider un node (évacuer les pods)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl taint nodes <node> key=value:NoSchedule    # Ajouter un taint
kubectl taint nodes <node> key:NoSchedule-          # Retirer un taint
kubectl label node <node> zone=eu          # Ajouter un label
```

---

## Pods

```bash
kubectl get pods                            # Lister les pods
kubectl get pods -o wide                    # Avec node et IP
kubectl get pods -A                         # Tous les namespaces
kubectl get pods --all-namespaces           # Idem
kubectl get pods -n <namespace>             # Namespace spécifique
kubectl get pods --show-labels              # Avec les labels
kubectl get pods -l app=nginx               # Filtrer par label
kubectl get pods --field-selector status.phase=Running
kubectl get pods -w                         # Watch (temps réel)

kubectl describe pod <pod>                  # Détails d'un pod
kubectl get pod <pod> -o yaml               # Définition YAML
kubectl get pod <pod> -o json               # Définition JSON

kubectl run nginx --image=nginx             # Créer un pod rapidement
kubectl run tmp --image=busybox -it --rm -- sh    # Pod éphémère interactif

kubectl delete pod <pod>                    # Supprimer un pod
kubectl delete pod <pod> --grace-period=0 --force   # Forcer
kubectl delete pods --all                   # Supprimer tous les pods
```

---

## Deployments

```bash
kubectl get deployments                     # Lister
kubectl get deploy -o wide                  # Détaillé
kubectl describe deployment <deploy>        # Détails

kubectl create deployment web --image=nginx           # Créer
kubectl create deployment web --image=nginx --replicas=3

kubectl edit deployment <deploy>            # Éditer en direct
kubectl delete deployment <deploy>          # Supprimer

kubectl set image deployment/web nginx=nginx:1.25     # Changer l'image
kubectl set env deployment/web KEY=val                # Variable d'env
kubectl set resources deployment/web --limits=cpu=200m,memory=512Mi
```

---

## ReplicaSets & DaemonSets

```bash
kubectl get replicasets                     # Lister les ReplicaSets
kubectl get rs                              # Abréviation
kubectl describe rs <rs>                    # Détails

kubectl get daemonsets                      # Lister les DaemonSets
kubectl get ds -A                           # Tous les namespaces
kubectl describe ds <ds>                    # Détails
kubectl rollout restart ds/<ds>             # Redémarrer
```

---

## Services

```bash
kubectl get services                        # Lister
kubectl get svc -o wide                     # Détaillé
kubectl describe svc <service>              # Détails

# Exposer un deployment
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose deployment web --type=NodePort --port=80
kubectl expose deployment web --type=LoadBalancer --port=80

kubectl delete svc <service>                # Supprimer
kubectl get endpoints                       # Endpoints des services
```

---

## Ingress

```bash
kubectl get ingress                         # Lister les ingress
kubectl get ing -A                          # Tous les namespaces
kubectl describe ingress <ingress>          # Détails
kubectl delete ingress <ingress>            # Supprimer
```

---

## Namespaces

```bash
kubectl get namespaces                      # Lister
kubectl get ns                              # Abréviation
kubectl create namespace <ns>               # Créer
kubectl delete namespace <ns>               # Supprimer
kubectl describe namespace <ns>             # Détails
kubectl get all -n <ns>                     # Toutes les ressources d'un ns
```

---

## ConfigMaps & Secrets

```bash
# ConfigMaps
kubectl get configmaps                      # Lister
kubectl create configmap myconf --from-literal=key=value
kubectl create configmap myconf --from-file=config.txt
kubectl create configmap myconf --from-env-file=.env
kubectl describe configmap myconf           # Détails
kubectl delete configmap myconf             # Supprimer

# Secrets
kubectl get secrets                         # Lister
kubectl create secret generic mysecret --from-literal=password=1234
kubectl create secret generic mysecret --from-file=./pass.txt
kubectl create secret docker-registry myreg --docker-server=... --docker-username=... --docker-password=...
kubectl create secret tls mytls --cert=cert.pem --key=key.pem
kubectl describe secret mysecret            # Détails
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 -d   # Décoder
kubectl delete secret mysecret              # Supprimer
```

---

## Volumes (PV / PVC)

```bash
kubectl get persistentvolumes               # Lister les PV
kubectl get pv                              # Abréviation
kubectl get persistentvolumeclaims          # Lister les PVC
kubectl get pvc                             # Abréviation
kubectl describe pv <pv>                    # Détails d'un PV
kubectl describe pvc <pvc>                  # Détails d'un PVC
kubectl delete pvc <pvc>                    # Supprimer
kubectl get storageclass                    # Classes de stockage
kubectl get sc                              # Abréviation
```

---

## Jobs & CronJobs

```bash
kubectl get jobs                            # Lister les jobs
kubectl describe job <job>                  # Détails
kubectl delete job <job>                    # Supprimer
kubectl create job myjob --image=busybox -- echo "Hello"

kubectl get cronjobs                        # Lister les cronjobs
kubectl get cj                              # Abréviation
kubectl create cronjob mycron --image=busybox --schedule="*/5 * * * *" -- echo hi
kubectl describe cronjob <cronjob>          # Détails
kubectl delete cronjob <cronjob>            # Supprimer
```

---

## StatefulSets

```bash
kubectl get statefulsets                    # Lister
kubectl get sts                             # Abréviation
kubectl describe sts <sts>                  # Détails
kubectl scale sts <sts> --replicas=3        # Mettre à l'échelle
kubectl rollout restart sts/<sts>           # Redémarrer
kubectl delete sts <sts>                    # Supprimer
```

---

## Logs & Debug

```bash
kubectl logs <pod>                          # Logs d'un pod
kubectl logs -f <pod>                       # Suivre en temps réel
kubectl logs <pod> -c <container>           # Conteneur spécifique
kubectl logs <pod> --previous               # Logs du conteneur précédent
kubectl logs <pod> --tail=100               # 100 dernières lignes
kubectl logs <pod> --since=10m              # Depuis 10 min
kubectl logs -l app=nginx                   # Logs par label

kubectl exec -it <pod> -- bash              # Shell dans un pod
kubectl exec -it <pod> -- sh                # Shell (Alpine)
kubectl exec <pod> -- ls /app               # Commande ponctuelle
kubectl exec -it <pod> -c <container> -- sh # Conteneur spécifique

kubectl cp <pod>:/chemin ./local            # Copier depuis le pod
kubectl cp ./local <pod>:/chemin            # Copier vers le pod

kubectl port-forward <pod> 8080:80          # Forward de port
kubectl port-forward svc/<service> 8080:80  # Forward d'un service

kubectl attach <pod> -it                    # S'attacher à un pod
kubectl debug <pod> -it --image=busybox     # Conteneur de debug éphémère
kubectl describe <ressource> <nom>          # Détails (events inclus)
```

---

## Apply / Create / Delete

```bash
kubectl apply -f fichier.yaml               # Appliquer (créer/mettre à jour)
kubectl apply -f ./dossier/                 # Tout un dossier
kubectl apply -f https://url/manifest.yaml  # Depuis une URL
kubectl apply -k ./kustomize/               # Kustomize

kubectl create -f fichier.yaml              # Créer (échoue si existe)
kubectl delete -f fichier.yaml              # Supprimer selon un manifest
kubectl replace -f fichier.yaml             # Remplacer

kubectl diff -f fichier.yaml                # Diff avant d'appliquer
kubectl apply -f fichier.yaml --dry-run=client    # Simulation
kubectl apply -f fichier.yaml --server-side       # Apply côté serveur

# Générer un YAML sans l'appliquer
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

---

## Scaling & Rollout

```bash
kubectl scale deployment web --replicas=5   # Mettre à l'échelle
kubectl scale --replicas=3 -f deploy.yaml   # Via un fichier

# Autoscaling (HPA)
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=80
kubectl get hpa                             # Lister les HPA

# Rollout
kubectl rollout status deployment/web       # État du déploiement
kubectl rollout history deployment/web      # Historique des révisions
kubectl rollout history deployment/web --revision=2   # Détail d'une révision
kubectl rollout undo deployment/web         # Annuler (revenir en arrière)
kubectl rollout undo deployment/web --to-revision=2   # Vers une révision
kubectl rollout restart deployment/web      # Redémarrer (recreate des pods)
kubectl rollout pause deployment/web        # Mettre en pause
kubectl rollout resume deployment/web       # Reprendre
```

---

## Labels & Annotations

```bash
kubectl label pod <pod> env=prod            # Ajouter un label
kubectl label pod <pod> env=prod --overwrite    # Écraser
kubectl label pod <pod> env-                # Supprimer un label
kubectl get pods -l env=prod                # Filtrer par label
kubectl get pods -l 'env in (prod,dev)'     # Sélecteur ensembliste

kubectl annotate pod <pod> description="mon pod"     # Ajouter une annotation
kubectl annotate pod <pod> description-              # Supprimer
```

---

## RBAC

```bash
kubectl get serviceaccounts                 # Lister les service accounts
kubectl get sa                              # Abréviation
kubectl create serviceaccount mysa          # Créer

kubectl get roles                           # Rôles (namespace)
kubectl get clusterroles                    # Rôles (cluster)
kubectl get rolebindings                    # Liaisons (namespace)
kubectl get clusterrolebindings             # Liaisons (cluster)

kubectl auth can-i create pods              # Vérifier une permission
kubectl auth can-i '*' '*'                  # Suis-je admin ?
kubectl auth can-i list secrets --as=system:serviceaccount:default:mysa
```

---

## Astuces

```bash
# Sortie et formats
kubectl get pods -o yaml                    # YAML complet
kubectl get pods -o json                    # JSON
kubectl get pods -o name                    # Noms seulement
kubectl get pod <pod> -o jsonpath='{.status.podIP}'    # Champ précis
kubectl get pods -o custom-columns=NOM:.metadata.name,STATUT:.status.phase

# Toutes les ressources
kubectl get all                             # Ressources courantes du ns
kubectl get all -A                          # Tous les namespaces

# Surveillance
kubectl get pods -w                         # Watch
watch kubectl get pods                      # Refresh continu

# Supprimer rapidement
kubectl delete pods --all -n <ns>
kubectl delete all --all -n <ns>

# Alias pratiques (à mettre dans ~/.bashrc ou ~/.zshrc)
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kaf='kubectl apply -f'
alias kdp='kubectl describe pod'

# Autocomplétion
source <(kubectl completion bash)           # Bash
source <(kubectl completion zsh)            # Zsh
```

---

## 📦 Exemple de manifest (Deployment + Service)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "500m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
