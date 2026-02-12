# 🗳️ MiniProjet CaaS — Pipeline CI/CD pour Docker Voting App

**Module :** Cloud as a Service (CaaS)  
**Projet :** Pipeline CI/CD — Docker Voting App (Microservices)  
**Date :** 12/02/2026  
**Étudiant :** Lopvet Lucas

---

## 📑 Table des matières

| Section | Contenu | Durée |
|---------|---------|:-----:|
| [0. Discussion](#0--discussion-sur-lidée-de-lapplication) | Architecture, langages, BDD | 1–2 min |
| [1. Démonstration](#1--démonstration-end-to-end) | Pipeline Jenkins → Docker Hub → K8s → Grafana | 7–8 min |
| [2. Justification](#2--justification--explication) | Jenkinsfile, Manifests K8s, Prometheus/Grafana | 3–4 min |
| [Annexe A](#annexe-a--installation-complète-sur-ubuntu) | Guide d'installation Ubuntu | — |
| [Annexe B](#annexe-b--dépannage) | Dépannage | — |

---

# 0 — Discussion sur l'idée de l'application

## Application choisie : Docker Voting App

Nous utilisons la **Docker Voting App**, application officielle de démonstration créée par Docker :  
🔗 [github.com/dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app)

C'est une application de **sondage en temps réel** : l'utilisateur vote pour **« Cats »** ou **« Dogs »**, et les résultats s'affichent instantanément.

## Architecture : Microservices (5 composants)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DOCKER VOTING APP                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌────────────────┐                           ┌────────────────┐          │
│   │   🗳️ VOTE      │                           │   📊 RESULT    │          │
│   │  (Python Flask) │                           │   (Node.js)    │          │
│   │  Port: 80       │                           │   Port: 80     │          │
│   └───────┬────────┘                           └───────┬────────┘          │
│           │                                            │                    │
│           │ écrit les votes                            │ lit les résultats  │
│           ▼                                            │                    │
│   ┌────────────────┐       ┌────────────────┐          │                    │
│   │   🔴 REDIS     │──────▶│   ⚙️ WORKER    │          │                    │
│   │   (Cache/Queue) │       │   (.NET Core)  │          │                    │
│   │   Port: 6379   │       │                │          │                    │
│   └────────────────┘       └───────┬────────┘          │                    │
│                                    │                    │                    │
│                                    │ écrit              │ lit                │
│                                    ▼                    ▼                    │
│                           ┌──────────────────────────────┐                  │
│                           │   🐘 POSTGRESQL               │                  │
│                           │   (Base de données)           │                  │
│                           │   Port: 5432                  │                  │
│                           └──────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Flux de données :**
```
Utilisateur ──▶ vote (Flask) ──▶ Redis ──▶ Worker (.NET) ──▶ PostgreSQL ──▶ result (Node.js) ──▶ Utilisateur
```

## Langages / Frameworks

| Service | Langage | Framework | Rôle |
|---------|---------|-----------|------|
| **vote** | Python | Flask + Gunicorn | Interface web de vote |
| **result** | Node.js | Express + Socket.io | Affichage des résultats en temps réel (WebSocket) |
| **worker** | C# (.NET 7) | — | Traitement des votes (Redis → PostgreSQL) |

## Bases de données — Justification du choix

| Base | Rôle | Pourquoi ce choix |
|------|------|-------------------|
| **Redis** | File d'attente temporaire (cache) | Ultra-rapide en mémoire, parfait pour du stockage temporaire de votes en attente. Structure de données `list` (RPUSH/LPOP) idéale pour une file d'attente |
| **PostgreSQL** | Stockage permanent des votes | Base relationnelle robuste, support ACID pour la persistance des données. Requêtes SQL pour l'agrégation des résultats (`COUNT`, `GROUP BY`) |

> **Pourquoi 2 bases ?** Redis sert de **tampon** entre le frontend (vote) et le backend (worker), ce qui découple l'écriture de la lecture. Le worker traite les votes à son rythme et les persiste dans PostgreSQL. Cela garantit la **résilience** (les votes ne sont pas perdus si le worker redémarre).

## Architecture CI/CD

```
┌─────────────┐     git push     ┌─────────────┐     webhook     ┌─────────────┐
│ Développeur │ ───────────────▶ │   GitHub    │ ──────────────▶ │   Jenkins   │
└─────────────┘                  └─────────────┘                 └──────┬──────┘
                                                                        │
                                       ┌────────────────────────────────┘
                                       │  1. Checkout
                                       │  2. Build images
                                       │  3. Push images
                                       ▼
                                ┌─────────────┐
                                │ Docker Hub  │
                                └──────┬──────┘
                                       │
                                       │ 4. kubectl apply
                                       ▼
                                ┌─────────────┐
                                │ Kubernetes  │ ◀── Prometheus + Grafana
                                │  (Minikube) │     (Monitoring)
                                └─────────────┘
```

## Structure du dépôt

```
MiniProjet_CaaS/
├── vote/                          # Microservice VOTE (Python Flask)
│   ├── Dockerfile
│   ├── app.py
│   ├── requirements.txt
│   ├── static/stylesheets/
│   └── templates/index.html
├── result/                        # Microservice RESULT (Node.js)
│   ├── Dockerfile
│   ├── server.js
│   ├── package.json
│   ├── views/
│   └── tests/
├── worker/                        # Microservice WORKER (.NET Core)
│   ├── Dockerfile
│   ├── Program.cs
│   └── Worker.csproj
├── k8s/                           # Manifests Kubernetes
│   ├── vote-deployment.yaml       # 2 replicas, 250m CPU, 128Mi RAM
│   ├── vote-service.yaml          # NodePort 31000
│   ├── result-deployment.yaml     # 1 replica, 250m CPU, 128Mi RAM
│   ├── result-service.yaml        # NodePort 31001
│   ├── worker-deployment.yaml     # 1 replica, 500m CPU, 256Mi RAM
│   ├── redis-deployment.yaml      # 1 replica
│   ├── redis-service.yaml         # ClusterIP
│   ├── db-deployment.yaml         # 1 replica, PostgreSQL 15
│   └── db-service.yaml            # ClusterIP
├── jenkins/                       # Jenkins dockerisé
│   ├── Dockerfile                 # Jenkins + Docker CLI + kubectl
│   └── docker-compose.yml         # Jenkins + DinD
├── Jenkinsfile                    # Pipeline CI/CD (4 stages)
└── Readme.md                     # Ce fichier
```

---

# 1 — Démonstration end-to-end

> **Durée : 7–8 minutes.** Cette section montre que toute la chaîne fonctionne.

## 1.1 — Lancer / Montrer la pipeline Jenkins

### Accéder à Jenkins

```bash
# Jenkins est accessible via le navigateur
# http://localhost:8080
```

Montrer :
1. Le **job `MiniProjet-CaaS`** dans le dashboard
2. Cliquer sur **Build Now** pour lancer un build (ou montrer le dernier run réussi)
3. Montrer la **Stage View** avec les 4 stages réussis (vert)
4. Cliquer sur le build → **Console Output** → montrer le résultat `SUCCESS`

> 📸 **CAPTURE D'ÉCRAN 1 :** Dashboard Jenkins avec le job MiniProjet-CaaS
>
> ![Jenkins Dashboard](image/capture_jenkins_dashboard.png)

> 📸 **CAPTURE D'ÉCRAN 2 :** Pipeline Jenkins — Stage View avec les 4 stages réussis
>
> ![Jenkins Pipeline](image/capture_jenkins_pipeline.png)

> 📸 **CAPTURE D'ÉCRAN 3 :** Console Output Jenkins — Résultat SUCCESS avec les logs
>
> ![Jenkins Console](image/capture_jenkins_console.png)

---

## 1.2 — Montrer les images sur Docker Hub

### Vérifier les images Docker Hub

```bash
# Ouvrir Docker Hub dans le navigateur :
# https://hub.docker.com/u/litlewolf

# Ou vérifier en ligne de commande :
docker images | grep litlewolf
```

Montrer sur Docker Hub pour chaque image (`litlewolf/vote`, `litlewolf/result`, `litlewolf/worker`) :
- Le **tag** (`latest` + numéro de build)
- La **date** de push
- Le **digest** (SHA256)

> 📸 **CAPTURE D'ÉCRAN 4 :** Page Docker Hub avec les 3 images (tag, date, digest visibles)
>
> ![Docker Hub Images](image/capture_dockerhub_images.png)

---

## 1.3 — Montrer Kubernetes

### Vérifier les pods, services et déploiements

```bash
# Vérifier que tous les pods tournent
kubectl get pods

# Résultat attendu : tous les pods en Running
# NAME                      READY   STATUS    RESTARTS   AGE
# db-xxxxx                  1/1     Running   0          XXm
# redis-xxxxx               1/1     Running   0          XXm
# vote-xxxxx                1/1     Running   0          XXm
# vote-yyyyy                1/1     Running   0          XXm  (2 replicas)
# result-xxxxx              1/1     Running   0          XXm
# worker-xxxxx              1/1     Running   0          XXm
```

```bash
# Vérifier les services exposés
kubectl get svc

# Résultat attendu :
# NAME    TYPE        CLUSTER-IP     PORT(S)
# vote    NodePort    10.x.x.x       5000:31000/TCP
# result  NodePort    10.x.x.x       5001:31001/TCP
# redis   ClusterIP   10.x.x.x       6379/TCP
# db      ClusterIP   10.x.x.x       5432/TCP
```

```bash
# Vérifier les déploiements
kubectl get deploy

# Résultat attendu :
# NAME     READY   UP-TO-DATE   AVAILABLE
# vote     2/2     2            2
# result   1/1     1            1
# worker   1/1     1            1
# redis    1/1     1            1
# db       1/1     1            1
```

> 📸 **CAPTURE D'ÉCRAN 5 :** Résultat de `kubectl get pods` — tous les pods en Running
>
> ![Kubectl Pods](image/capture_kubectl_pods.png)

> 📸 **CAPTURE D'ÉCRAN 6 :** Résultat de `kubectl get svc` — services avec leurs ports
>
> ![Kubectl Services](image/capture_kubectl_services.png)

> 📸 **CAPTURE D'ÉCRAN 7 :** Résultat de `kubectl get deploy` — tous les déploiements READY
>
> ![Kubectl Deployments](image/capture_kubectl_deploy.png)

### Accéder à l'application

```bash
# Obtenir l'URL du service vote
minikube service vote --url
# → Ouvre http://<minikube-ip>:31000

# Obtenir l'URL du service result
minikube service result --url
# → Ouvre http://<minikube-ip>:31001
```

| Service | URL |
|---------|-----|
| **Vote** (voter) | `http://<MINIKUBE_IP>:31000` |
| **Result** (résultats) | `http://<MINIKUBE_IP>:31001` |

**Démonstration live :**
1. Ouvrir l'interface **vote** → voter pour « Cats » ou « Dogs »
2. Ouvrir l'interface **result** → constater que le vote apparaît **en temps réel**
3. Voter plusieurs fois → les pourcentages changent en direct

> 📸 **CAPTURE D'ÉCRAN 8 :** Interface de vote — page web Cats vs Dogs
>
> ![Interface Vote](image/capture_vote_app.png)

> 📸 **CAPTURE D'ÉCRAN 9 :** Page des résultats — mise à jour en temps réel
>
> ![Interface Result](image/capture_result_app.png)

---

## 1.4 — Montrer le monitoring (Grafana)

### Accéder à Grafana

```bash
# Exposer Grafana via port-forward (si pas déjà fait)
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
```

Ouvrir : **http://localhost:3000** (login: `admin` / `admin`)

### Dashboard à montrer

Aller dans **Dashboards** et montrer :
- **CPU par pod** dans le namespace `default`
- **Mémoire par pod**
- **Nombre de pods en running**

> 📸 **CAPTURE D'ÉCRAN 10 :** Dashboard Grafana — métriques CPU/mémoire des pods de l'application
>
> ![Grafana Dashboard](image/capture_grafana_dashboard.png)

> 📸 **CAPTURE D'ÉCRAN 11 :** Prometheus — requête exécutée (optionnel)
>
> ![Prometheus](image/capture_prometheus.png)

---

# 2 — Justification / explication

> **Durée : 3–4 minutes.** Expliquer rapidement les fichiers de configuration clés.

## 2.1 — Le Jenkinsfile

Le fichier `Jenkinsfile` à la racine du projet définit la pipeline CI/CD en **4 stages** :

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'litlewolf'
        VOTE_IMAGE     = "${DOCKERHUB_USER}/vote"
        RESULT_IMAGE   = "${DOCKERHUB_USER}/result"
        WORKER_IMAGE   = "${DOCKERHUB_USER}/worker"
        BUILD_TAG      = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout')           { ... } // 1. Récupération du code depuis GitHub
        stage('Build Docker Images') { ... } // 2. Construction des 3 images Docker
        stage('Push to Docker Hub')  { ... } // 3. Push vers Docker Hub avec credentials
        stage('Deploy to Kubernetes'){ ... } // 4. Déploiement via kubectl apply -f k8s/
    }
}
```

### Explication des points clés

| Élément | Explication |
|---------|------------|
| **`agent any`** | La pipeline peut s'exécuter sur n'importe quel agent Jenkins disponible |
| **`environment`** | Variables globales : user Docker Hub, noms d'images, tag = numéro de build |
| **`BUILD_TAG`** | Chaque build génère un tag unique (ex: `litlewolf/vote:3`) + `latest` |
| **`withCredentials`** | Utilise les credentials `dockerhub-credentials` stockés dans Jenkins pour le `docker login` |
| **`kubectl apply -f k8s/`** | Applique tous les manifests Kubernetes d'un coup |
| **`kubectl rollout status`** | Attend que chaque déploiement soit terminé avant de continuer |

### Sécurité
- Les identifiants Docker Hub sont stockés dans Jenkins (pas dans le code)
- Le `docker logout` est fait systématiquement dans le bloc `post > always`

---

## 2.2 — Les manifests Kubernetes

Les 9 fichiers dans `k8s/` définissent **5 Deployments** et **4 Services** :

### Déploiements

| Deployment | Image | Replicas | CPU | Mémoire | Pourquoi |
|------------|-------|:--------:|:---:|:-------:|----------|
| **vote** | `litlewolf/vote:latest` | **2** | 250m | 128Mi | 2 replicas pour la haute disponibilité (frontend principal) |
| **result** | `litlewolf/result:latest` | 1 | 250m | 128Mi | 1 replica suffit (affichage uniquement) |
| **worker** | `litlewolf/worker:latest` | 1 | 500m | 256Mi | Plus de ressources car traitement de données (Redis → PostgreSQL) |
| **redis** | `redis:alpine` | 1 | 250m | 128Mi | Image officielle légère Alpine |
| **db** | `postgres:15-alpine` | 1 | 500m | 256Mi | PostgreSQL 15 avec `emptyDir` pour le volume (données éphémères pour la démo) |

### Services

| Service | Type | Port externe | Port interne | Pourquoi |
|---------|------|:------------:|:------------:|----------|
| **vote** | **NodePort** | 31000 | 80 | Accessible depuis l'extérieur du cluster pour voter |
| **result** | **NodePort** | 31001 | 80 | Accessible depuis l'extérieur pour voir les résultats |
| **redis** | ClusterIP | — | 6379 | Accessible uniquement au sein du cluster (communication interne) |
| **db** | ClusterIP | — | 5432 | Accessible uniquement au sein du cluster (communication interne) |

> **Pourquoi NodePort ?** Minikube ne supporte pas le type `LoadBalancer` nativement. NodePort expose un port fixe sur l'IP du nœud, accessible via `minikube service`.

> **Pas d'Ingress** car pour une démo locale avec Minikube, NodePort est suffisant et plus simple à mettre en place.

---

## 2.3 — Prometheus / Grafana : comment c'est branché

### Installation via Helm

```bash
# Le chart kube-prometheus-stack installe tout d'un coup :
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin
```

### Ce que le chart installe

| Composant | Rôle |
|-----------|------|
| **Prometheus** | Collecte les métriques — scrape automatiquement tous les pods/nodes Kubernetes |
| **Grafana** | Visualisation — préconfigurée avec Prometheus comme datasource |
| **AlertManager** | Alertes (non utilisé dans cette démo) |
| **kube-state-metrics** | Expose les métriques des objets Kubernetes (pods, deployments, services) |
| **node-exporter** | Expose les métriques système des nœuds (CPU, RAM, disque) |

### Comment le scrape fonctionne

```
┌──────────────┐     scrape /metrics     ┌──────────────────┐
│  Prometheus  │ ◀────────────────────── │ kube-state-metrics│  (métriques K8s)
│              │ ◀────────────────────── │ node-exporter     │  (métriques système)
│              │ ◀────────────────────── │ kubelet/cAdvisor  │  (métriques conteneurs)
└──────┬───────┘                         └──────────────────┘
       │
       │ datasource auto-configurée
       ▼
┌──────────────┐
│   Grafana    │ → Dashboards pré-installés + import de dashboards personnalisés
└──────────────┘
```

- **Prometheus** scrape automatiquement les endpoints grâce aux `ServiceMonitor` CRDs installés par le chart Helm
- **Grafana** est préconfigurée avec la datasource Prometheus — pas de configuration manuelle nécessaire
- Les dashboards importés (ID `15661`, `6417`, `315`) fournissent des vues prêtes à l'emploi

### Requêtes Prometheus utiles

```promql
# Utilisation CPU par pod (namespace default = nos pods applicatifs)
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Mémoire utilisée par pod
sum(container_memory_usage_bytes{namespace="default"}) by (pod)

# Nombre de pods en running
count(kube_pod_status_phase{phase="Running", namespace="default"})
```

---

# Annexe A — Installation complète sur Ubuntu

> **Toutes les commandes ci-dessous sont pour Ubuntu (22.04 LTS recommandé).**  
> La machine doit disposer d'au moins **8 Go de RAM** et **4 CPU**.

## A.1 — Prérequis (installation des outils)

### Docker

```bash
# Installer Docker
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker $USER
newgrp docker

# Vérifier
docker --version
```

### Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Vérifier
minikube version
```

### kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Vérifier
kubectl version --client
```

### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Vérifier
helm version
```

### Vérification rapide de tous les outils

```bash
git --version
docker --version
minikube version
kubectl version --client
helm version
```

> 📸 **CAPTURE D'ÉCRAN :** Résultat de la vérification des outils (toutes les versions affichées)
>
> ![Vérification des outils](image/capture_outils.png)

---

## A.2 — Cloner le dépôt

```bash
git clone https://github.com/LittleWolf-Code/MiniProjet_CaaS.git
cd MiniProjet_CaaS
git checkout projet-final
ls -la
```

---

## A.3 — Démarrer Minikube

```bash
# Démarrer le cluster Minikube
minikube start --driver=docker --cpus=4 --memory=4096

# Vérifier
kubectl cluster-info
kubectl get nodes
```

---

## A.4 — Construire et pousser les images Docker (optionnel, si pas de Jenkins)

> ⚠️ **Remplacez `litlewolf` par votre identifiant Docker Hub si différent.**

```bash
export DOCKERHUB_USER="litlewolf"

# Build des 3 images
docker build -t $DOCKERHUB_USER/vote:latest ./vote
docker build -t $DOCKERHUB_USER/result:latest ./result
docker build -t $DOCKERHUB_USER/worker:latest ./worker

# Push vers Docker Hub
docker login
docker push $DOCKERHUB_USER/vote:latest
docker push $DOCKERHUB_USER/result:latest
docker push $DOCKERHUB_USER/worker:latest
```

---

## A.5 — Installer Jenkins (dockerisé)

```bash
# Créer le réseau minikube si nécessaire
docker network create minikube 2>/dev/null || true

# Lancer Jenkins + DinD avec docker compose
cd jenkins
docker compose up -d
cd ..

# Récupérer le mot de passe admin initial
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

Accéder à Jenkins via : **http://localhost:8080**

### Configurer Jenkins

1. **Installer les plugins suggérés** (lors de la première connexion)
2. **Ajouter les credentials Docker Hub :**
   - Aller dans **Manage Jenkins** → **Credentials** → **(global)** → **Add Credentials**
   - Kind : `Username with password`
   - Username : votre identifiant Docker Hub
   - Password : votre mot de passe Docker Hub
   - ID : `dockerhub-credentials`

3. **Créer la pipeline :**
   - **New Item** → Nom : `MiniProjet-CaaS` → Type : **Pipeline**
   - Pipeline → Definition : `Pipeline script from SCM`
   - SCM : Git
   - Repository URL : `https://github.com/LittleWolf-Code/MiniProjet_CaaS.git`
   - Branch : `*/projet-final`
   - Script Path : `Jenkinsfile`
   - **Save**

4. **Lancer le build :** Cliquer sur **Build Now**

---

## A.6 — Déployer sur Kubernetes

```bash
# Appliquer tous les manifests
kubectl apply -f k8s/

# Attendre que tous les pods soient prêts
kubectl rollout status deployment/vote
kubectl rollout status deployment/result
kubectl rollout status deployment/worker
kubectl rollout status deployment/redis
kubectl rollout status deployment/db

# Vérifier
kubectl get pods
kubectl get svc
kubectl get deploy
```

### Accéder à l'application

```bash
minikube service vote --url
# → http://<minikube-ip>:31000

minikube service result --url
# → http://<minikube-ip>:31001
```

---

## A.7 — Installer le monitoring (Prometheus + Grafana)

```bash
# Ajouter le repo Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Installer la stack complète
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin

# Attendre que les pods soient prêts (~2-3 minutes)
kubectl get pods -n monitoring -w
```

### Accéder à Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
```

Ouvrir : **http://localhost:3000** (login : `admin` / `admin`)

### Importer un dashboard

1. **Dashboards** → **Import**
2. Entrer l'ID : **`15661`** (Kubernetes Cluster Monitoring)
3. **Load** → sélectionner la datasource **Prometheus** → **Import**

| ID | Dashboard recommandé |
|----|--------------------|
| **15661** | Kubernetes Cluster Monitoring |
| **6417** | Kubernetes Pods Monitoring |
| **315** | Kubernetes Cluster Overview |

### Accéder à Prometheus (optionnel)

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
```

Ouvrir : **http://localhost:9090**

---

# Annexe B — Dépannage

### Pods en CrashLoopBackOff

```bash
kubectl logs <nom-du-pod>
kubectl describe pod <nom-du-pod>
```

### Minikube ne démarre pas

```bash
minikube delete
minikube start --driver=docker --cpus=4 --memory=4096
```

### Docker permission denied

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Jenkins ne peut pas accéder à Docker

```bash
# Si Jenkins dockerisé : vérifier que DinD tourne
docker ps | grep jenkins-docker

# Si Jenkins natif :
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Réinitialiser le déploiement Kubernetes

```bash
kubectl delete -f k8s/
kubectl apply -f k8s/
```

---

## 🌐 Résumé des ports et accès

| Service | Type | Port | URL |
|---------|------|:----:|-----|
| **Vote** (frontend) | NodePort | 31000 | `http://<MINIKUBE_IP>:31000` |
| **Result** (frontend) | NodePort | 31001 | `http://<MINIKUBE_IP>:31001` |
| **Redis** | ClusterIP | 6379 | Interne uniquement |
| **PostgreSQL** | ClusterIP | 5432 | Interne uniquement |
| **Jenkins** | Docker | 8080 | `http://localhost:8080` |
| **Grafana** | Port-forward | 3000 | `http://localhost:3000` |
| **Prometheus** | Port-forward | 9090 | `http://localhost:9090` |

```bash
# Récupérer l'IP de Minikube
minikube ip
```

---

## 🎯 Conclusion

Ce projet démontre la mise en place d'une **chaîne DevOps complète** :

| Étape | Réalisation |
|-------|------------|
| **1. Code source** | ✅ Repository GitHub avec code source structuré (3 microservices) |
| **2. Dockerisation** | ✅ 3 images Docker (vote, result, worker) construites et poussées sur Docker Hub |
| **3. CI/CD** | ✅ Pipeline Jenkins automatisée (Checkout → Build → Push → Deploy) |
| **4. Orchestration** | ✅ Déploiement Kubernetes avec 5 services sur Minikube |
| **5. Monitoring** | ✅ Prometheus + Grafana pour la supervision du cluster |

**Technologies utilisées :** Git, GitHub, Docker, Docker Hub, Jenkins, Kubernetes (Minikube), Helm, Prometheus, Grafana

---

> **Auteur :** Lopvet Lucas  
> **Module :** Cloud as a Service (CaaS)  
> **Date :** 12/02/2026