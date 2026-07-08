# Mettre en place le CI/CD pour getting-started-app

## Étape 1 — Créer un compte Docker Hub (gratuit)
1. Va sur https://hub.docker.com et crée un compte (si pas déjà fait)
2. Une fois connecté : Account Settings → Personal access tokens → **Generate new token**
   - Donne-lui un nom (ex: `github-actions`)
   - Permissions : `Read & Write`
   - **Copie le token affiché** (tu ne pourras plus le revoir après)

## Étape 2 — Préparer le repo GitHub
Ton code (`Dockerfile`, `src/`, `package.json`, `k8s/`) doit être dans un repo GitHub. Si `getting-started-app` n'est pas encore sur GitHub :

```powershell
cd C:\Users\wylso\Downloads\getting-started-app-main\getting-started-app-main
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/wylson02/getting-started-app.git
git push -u origin main
```

(Crée d'abord le repo vide `getting-started-app` sur GitHub si besoin, via le bouton "New repository".)

Copie ensuite le dossier `k8s/` (avec le fichier `04-app-deployment.yaml` mis à jour que je t'ai donné) et le fichier `.github/workflows/ci-cd.yml` dans ce même repo, puis commit/push.

**N'oublie pas** de remplacer `TON_USERNAME_DOCKERHUB` dans `k8s/04-app-deployment.yaml` par ton vrai nom d'utilisateur Docker Hub.

## Étape 3 — Configurer les secrets GitHub
Sur GitHub : ton repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Ajoute :
- `DOCKERHUB_USERNAME` : ton nom d'utilisateur Docker Hub
- `DOCKERHUB_TOKEN` : le token généré à l'étape 1

## Étape 4 — Installer un runner GitHub Actions auto-hébergé sur ton PC
C'est la partie qui permet à GitHub de déployer sur **ton** cluster local.

Sur GitHub : ton repo → **Settings** → **Actions** → **Runners** → **New self-hosted runner** → choisis **Windows**

GitHub te donne des commandes PowerShell à copier-coller, du style :

```powershell
mkdir actions-runner ; cd actions-runner
Invoke-WebRequest -Uri <URL fournie par GitHub> -OutFile actions-runner.zip
Expand-Archive -Path actions-runner.zip -DestinationPath .
.\config.cmd --url https://github.com/wylson02/getting-started-app --token <TOKEN fourni par GitHub>
.\run.cmd
```

**Laisse cette fenêtre PowerShell ouverte** — c'est elle qui écoute GitHub et exécute le déploiement quand tu push du code. (Tu peux aussi l'installer comme service Windows pour qu'il tourne en arrière-plan, GitHub explique comment sur la même page.)

## Étape 5 — Tester
Fais une petite modif dans ton code (ex: change un texte dans `src/`), puis :

```powershell
git add .
git commit -m "test CI/CD"
git push
```

Va dans l'onglet **Actions** de ton repo GitHub : tu dois voir le workflow se déclencher, construire l'image, la pousser sur Docker Hub, puis (grâce au runner sur ton PC) redéployer automatiquement sur ton cluster.

Vérifie ensuite que le pod a bien été mis à jour :

```powershell
kubectl rollout history deployment/app
kubectl get pods
```

## Résumé du flux

```
git push  →  GitHub Actions (cloud) build + push image sur Docker Hub
                        ↓
          Runner sur TON PC récupère le job
                        ↓
          kubectl apply + kubectl set image
                        ↓
          Ton cluster local redéploie la nouvelle version
```
