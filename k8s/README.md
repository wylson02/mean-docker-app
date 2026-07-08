# Déployer l'app sur Kubernetes (Docker Desktop)

## 0. Réparer le cluster (si besoin)
Si `kubectl get nodes` échoue ou renvoie des erreurs `memcache.go` :
Docker Desktop → ⚙️ Settings → Kubernetes → **Reset Kubernetes Cluster**, puis attends que le statut passe à "Active".

## 1. Construire l'image Docker localement
Depuis le dossier qui contient le `Dockerfile` :

```bash
docker build -t getting-started .
```

Le cluster Kubernetes de Docker Desktop partage le même moteur Docker, donc pas besoin de push l'image sur un registry : `imagePullPolicy: IfNotPresent` suffit pour utiliser l'image locale.

## 2. Appliquer les manifests

```bash
kubectl apply -f k8s/
```

Les fichiers sont préfixés par un numéro pour garantir l'ordre (secret → volume → mysql → app).

## 3. Vérifier que tout tourne

```bash
kubectl get pods
kubectl get svc
```

Attends que les deux pods (`mysql-...` et `app-...`) soient en `Running` (le pod `app` a un initContainer qui attend que MySQL soit prêt, ça peut prendre 15-30s).

## 4. Accéder à l'app

```
http://localhost:30000
```

## 5. Nettoyer

```bash
kubectl delete -f k8s/
```

## Équivalences avec ton compose.yaml

| compose.yaml              | Kubernetes                                  |
|----------------------------|----------------------------------------------|
| `services.app`             | Deployment `app` + Service `app` (NodePort)  |
| `services.mysql`           | Deployment `mysql` + Service `mysql`         |
| `environment:`             | `env:` + Secret `mysql-secret`               |
| `depends_on: condition: service_healthy` | `initContainer` qui attend le port 3306 |
| `volumes: todo-mysql-data` | `PersistentVolumeClaim`                      |
| `ports: 3000:3000`         | Service `NodePort` (port 30000)              |
| `healthcheck:`             | `readinessProbe` / `livenessProbe`           |
