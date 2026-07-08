# getting-started-app — Récap Kubernetes + CI/CD + Autoscaling

Ce README résume toutes les manipulations à refaire après un redémarrage du PC, et les commandes utiles pour tester chaque brique (déploiement, CI/CD, autoscaling).

---

## 1. Démarrage après un reboot

### 1.1 Vérifier Docker Desktop + Kubernetes
Ouvre Docker Desktop → l'onglet **Kubernetes** doit afficher **Active**.
Si erreurs `memcache.go` ou cluster bloqué :
Docker Desktop → ⚙️ Settings → Kubernetes → **Reset Kubernetes Cluster**

### 1.2 Vérifier l'état des pods
```powershell
kubectl get pods
```
Les pods `app-...` et `mysql-...` doivent être `1/1 Running`.

Si un pod est bloqué en `Unknown` (fréquent après un reboot brutal) :
```powershell
kubectl delete pod <nom-du-pod> --force --grace-period=0
```
Le Deployment le recrée automatiquement.

### 1.3 Relancer le runner self-hosted (CI/CD)
Sans ça, un `git push` ne déclenchera jamais le déploiement automatique.
```powershell
cd C:\Users\wylso\Downloads\getting-started-app-k8s\actions-runner
.\run.cmd
```
Attendre le message **"Listening for Jobs"**. Laisser cette fenêtre ouverte tant que tu veux que le CD fonctionne.

> 💡 Pour ne plus avoir à faire ça manuellement : installer le runner comme service Windows une bonne fois pour toutes :
> ```powershell
> .\svc.cmd install
> .\svc.cmd start
> ```
> Il tournera alors en arrière-plan, même après redémarrage.

### 1.4 Accéder à l'app dans le navigateur
```powershell
kubectl port-forward service/app 3000:3000
```
Puis va sur **http://localhost:3000**
(Cette commande bloque le terminal — ouvre-en un nouveau pour la suite, et il faut la relancer si le pod `app` est recréé, ex: après un déploiement CD ou un scaling.)

---

## 2. Commandes Kubernetes utiles au quotidien

```powershell
kubectl get pods                        # état des pods
kubectl get svc                         # état des services (ports exposés)
kubectl get hpa                         # état de l'autoscaler
kubectl logs deployment/app             # logs de l'app
kubectl logs deployment/mysql           # logs de MySQL
kubectl describe pod <nom-du-pod>       # détails + événements d'un pod (utile pour debug)
kubectl rollout history deployment/app  # historique des déploiements
kubectl rollout restart deployment/app  # forcer un redéploiement manuel
```

Pour tout réappliquer depuis zéro (après modif des fichiers `k8s/`) :
```powershell
cd C:\Users\wylso\Downloads\getting-started-app-k8s
kubectl apply -f k8s/
```

Pour tout supprimer proprement :
```powershell
kubectl delete -f k8s/
```

---

## 3. Tester le pipeline CI/CD

1. Vérifie que le runner tourne (**"Listening for Jobs"**, voir 1.3)
2. Modifie un fichier du code (ex: un texte dans `src/`)
3. Push :
   ```powershell
   cd C:\Users\wylso\Downloads\getting-started-app-k8s
   git add .
   git commit -m "test"
   git push
   ```
4. Va sur https://github.com/wylson02/mean-docker-app/actions
   → le workflow **"CI/CD - getting-started-app"** doit se lancer, avec deux jobs verts : `build-and-push` puis `deploy`
5. Vérifie que le pod `app` a été recréé (nouvel `AGE` proche de 0) :
   ```powershell
   kubectl get pods
   ```
6. Relance le port-forward (voir 1.4) et rafraîchis `localhost:3000` pour voir le changement

Pour relancer un run manuellement sans nouveau commit :
```powershell
git commit --allow-empty -m "retest"
git push
```
ou depuis GitHub : onglet **Actions** → run concerné → **Re-run jobs**.

---

## 4. Tester l'autoscaling (HPA)

### 4.1 Vérifier que metrics-server tourne
```powershell
kubectl top pods
```
Doit afficher des valeurs CPU/mémoire (pas une erreur).

### 4.2 Générer de la charge artificielle
```powershell
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://app:3000; done"
```

### 4.3 Observer le scaling en direct (dans un autre terminal)
```powershell
kubectl get hpa --watch
```
Tu dois voir le `%CPU` grimper au-dessus de 50%, puis `REPLICAS` augmenter progressivement jusqu'à 5.

### 4.4 Arrêter la charge
`Ctrl+C` sur le terminal du `load-generator` (ou attends le `--rm` qui nettoie le pod tout seul à l'arrêt).

Après ~5 minutes (fenêtre de stabilisation par défaut de Kubernetes, pour éviter les oscillations), les replicas redescendent automatiquement jusqu'au minimum (1).

---

## 5. Arborescence du projet

```
getting-started-app-k8s/
├── Dockerfile
├── compose.yaml
├── package.json / package-lock.json
├── src/                        # code source de l'app
├── README-CICD.md              # setup détaillé du CI/CD (Docker Hub, secrets, runner)
├── .github/workflows/ci-cd.yml # pipeline GitHub Actions
├── actions-runner/              # runner self-hosted (ne pas commiter, voir .gitignore)
└── k8s/
    ├── 00-mysql-secret.yaml
    ├── 01-mysql-pvc.yaml
    ├── 02-mysql-deployment.yaml
    ├── 03-mysql-service.yaml
    ├── 04-app-deployment.yaml   # image Docker Hub + resources CPU/RAM
    ├── 05-app-service.yaml
    └── 06-app-hpa.yaml          # autoscaler (1 à 5 replicas, seuil 50% CPU)
```

---

## 6. Pense-bête des pièges déjà rencontrés

- **`docker build` échoue "Dockerfile not found"** → vérifier qu'on est bien dans le bon dossier avec `dir` avant de lancer la commande.
- **`localhost:30000` (NodePort) ne répond pas** → normal avec Docker Desktop, utiliser `kubectl port-forward` sur le port 3000, ou passer le Service en `LoadBalancer` :
  ```powershell
  kubectl patch service app -p "{\"spec\": {\"type\": \"LoadBalancer\"}}"
  ```
- **Le port-forward se coupe après un déploiement** → normal, le pod ciblé a été remplacé. Relancer la commande.
- **Pod bloqué en `Unknown` après reboot** → `kubectl delete pod <nom> --force --grace-period=0`
- **Workflow GitHub échoue "Username and password required"** → le secret `DOCKERHUB_TOKEN` est invalide/expiré, en régénérer un sur hub.docker.com et refaire `gh secret set DOCKERHUB_TOKEN --repo ... --body "..."`.
- **Token/secret visible dans une capture d'écran** → le considérer comme compromis, le révoquer et en générer un nouveau immédiatement.
