# 🗳️ MiniProjet CaaS — Pipeline CI/CD pour Docker Voting App

**Module :** Cloud as a Service (CaaS)  
**Projet :** Pipeline CI/CD — Docker Voting App (Microservices)  
**Date :** 12/02/2026  
**Étudiant :** Lopvet Lucas

---

## 📑 Table des matières

| Section | Contenu | Durée soutenance |
|---------|---------|:-----:|
| [0. Discussion](#0--discussion-sur-lidée-de-lapplication) | Architecture, langages, BDD | 1–2 min |
| [1. Démonstration](#1--démonstration-end-to-end) | Pipeline Jenkins → Docker Hub → K8s → Grafana | 7–8 min |
| [2. Justification](#2--justification--explication) | Jenkinsfile, Manifests K8s, Prometheus/Grafana | 3–4 min |
| [Guide de réalisation](#-guide-de-réalisation-pas-à-pas) | Toutes les commandes depuis le git clone | — |

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
| **Redis** | File d'attente temporaire (cache) | Ultra-rapide en mémoire, parfait pour du stockage temporaire de votes en attente. Structure `list` (RPUSH/LPOP) idéale pour une file d'attente |
| **PostgreSQL** | Stockage permanent des votes | Base relationnelle robuste, support ACID pour la persistance. Requêtes SQL pour l'agrégation des résultats (`COUNT`, `GROUP BY`) |

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

---

# 1 — Démonstration end-to-end

> **Durée : 7–8 minutes.** Cette section montre que toute la chaîne fonctionne.

## 1.1 — Lancer / Montrer la pipeline Jenkins

Accéder à Jenkins : **http://localhost:8080**

Montrer :
1. Le **job `MiniProjet-CaaS`** dans le dashboard
2. Cliquer sur **Build Now** pour lancer un build (ou montrer le dernier run réussi)
3. Montrer la **Stage View** avec les 4 stages réussis (vert)
4. Cliquer sur le build → **Console Output** → montrer le résultat `SUCCESS`

> 📸 **CAPTURE 1 :** Dashboard Jenkins avec le job
>
> ![Jenkins Dashboard](image/capture_jenkins_dashboard.png)

> 📸 **CAPTURE 2 :** Stage View — 4 stages réussis
>
> ![Jenkins Pipeline](image/capture_jenkins_pipeline.png)

> 📸 **CAPTURE 3 :** Console Output — SUCCESS
>
> ![Jenkins Console](image/capture_jenkins_console.png)

---

## 1.2 — Montrer les images sur Docker Hub

```bash
# Ouvrir https://hub.docker.com/u/litlewolf
# Ou vérifier en ligne de commande :
docker images | grep litlewolf
```

Montrer pour chaque image (`litlewolf/vote`, `litlewolf/result`, `litlewolf/worker`) :
- Le **tag** (`latest` + numéro de build)
- La **date** de push
- Le **digest** (SHA256)

> 📸 **CAPTURE 4 :** Docker Hub — les 3 images (tag, date, digest)
>
> ![Docker Hub](image/capture_dockerhub_images.png)

---

## 1.3 — Montrer Kubernetes

```bash
kubectl get pods
kubectl get svc
kubectl get deploy
```

Résultats attendus : tous les pods en **Running**, 2 replicas pour vote, services NodePort sur 31000 et 31001.

```bash
# Accéder à l'application
minikube service vote --url      # → http://<IP>:31000
minikube service result --url    # → http://<IP>:31001
```

**Démonstration live :** Voter pour Cats/Dogs → voir les résultats en temps réel.

> 📸 **CAPTURE 5 :** `kubectl get pods` — tous en Running
>
> ![Kubectl Pods](image/capture_kubectl_pods.png)

> 📸 **CAPTURE 6 :** `kubectl get svc` + `kubectl get deploy`
>
> ![Kubectl Services](image/capture_kubectl_services.png)

> 📸 **CAPTURE 7 :** Interface de vote Cats vs Dogs
>
> ![Vote App](image/capture_vote_app.png)

> 📸 **CAPTURE 8 :** Page des résultats en temps réel
>
> ![Result App](image/capture_result_app.png)

---

## 1.4 — Montrer le monitoring (Grafana)

Accéder à Grafana : **http://localhost:3000** (login : `admin` / `admin`)

Montrer les dashboards avec métriques CPU / mémoire des pods.

> 📸 **CAPTURE 9 :** Dashboard Grafana — métriques CPU/mémoire
>
> ![Grafana](image/capture_grafana_dashboard.png)

---

# 2 — Justification / explication

> **Durée : 3–4 minutes.**

## 2.1 — Le Jenkinsfile

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
        stage('Checkout')            { ... } // 1. Récupère le code depuis GitHub
        stage('Build Docker Images') { ... } // 2. Build les 3 images Docker
        stage('Push to Docker Hub')  { ... } // 3. Push avec credentials Jenkins
        stage('Deploy to Kubernetes'){ ... } // 4. kubectl apply -f k8s/
    }
}
```

| Élément | Explication |
|---------|------------|
| **`agent any`** | S'exécute sur n'importe quel agent Jenkins |
| **`BUILD_TAG`** | Tag unique par build (ex: `litlewolf/vote:3`) + `latest` |
| **`withCredentials`** | Credentials `dockerhub-credentials` stockés dans Jenkins (pas en clair) |
| **`kubectl apply -f k8s/`** | Applique tous les manifests K8s d'un coup |
| **`kubectl rollout status`** | Attend la fin du déploiement avant de passer au suivant |
| **`docker logout`** | Toujours exécuté (bloc `post > always`) pour la sécurité |

---

## 2.2 — Les manifests Kubernetes

### Déploiements

| Deployment | Image | Replicas | CPU | Mémoire | Pourquoi |
|------------|-------|:--------:|:---:|:-------:|----------|
| **vote** | `litlewolf/vote:latest` | **2** | 250m | 128Mi | 2 replicas pour la haute disponibilité (frontend) |
| **result** | `litlewolf/result:latest` | 1 | 250m | 128Mi | 1 replica suffit (lecture seule) |
| **worker** | `litlewolf/worker:latest` | 1 | 500m | 256Mi | Plus de ressources (traitement Redis → PG) |
| **redis** | `redis:alpine` | 1 | 250m | 128Mi | Image officielle Alpine (légère) |
| **db** | `postgres:15-alpine` | 1 | 500m | 256Mi | `emptyDir` pour le volume (démo) |

### Services

| Service | Type | Port externe | Pourquoi |
|---------|------|:------------:|----------|
| **vote** | **NodePort** | 31000 | Accessible depuis l'extérieur pour voter |
| **result** | **NodePort** | 31001 | Accessible pour voir les résultats |
| **redis** | ClusterIP | — | Communication interne uniquement |
| **db** | ClusterIP | — | Communication interne uniquement |

> **NodePort** car Minikube ne supporte pas `LoadBalancer` nativement. Pas d'Ingress car NodePort suffit pour une démo locale.

---

## 2.3 — Prometheus / Grafana

### Installation via Helm (kube-prometheus-stack)

Le chart Helm `kube-prometheus-stack` installe **tout automatiquement** :

| Composant | Rôle |
|-----------|------|
| **Prometheus** | Scrape automatiquement tous les pods/nodes K8s |
| **Grafana** | Préconfigurée avec Prometheus comme datasource |
| **kube-state-metrics** | Métriques des objets K8s (pods, deploys) |
| **node-exporter** | Métriques système des nœuds (CPU, RAM) |

### Comment le scrape fonctionne

```
Prometheus ◀── scrape ── kube-state-metrics  (métriques K8s)
           ◀── scrape ── node-exporter       (métriques système)
           ◀── scrape ── kubelet/cAdvisor    (métriques conteneurs)
     │
     ▼ datasource auto-configurée
  Grafana → Dashboards pré-installés + import ID 15661/6417/315
```

- Les `ServiceMonitor` CRDs configurent automatiquement les cibles de scrape
- Grafana est préconfigurée — aucune configuration manuelle nécessaire
- Dashboards importés par ID pour des vues prêtes à l'emploi

---

# 🚀 Guide de réalisation pas à pas

> **Ce guide suppose une machine Ubuntu avec les outils suivants déjà installés :**  
> Git, Docker, Minikube, kubectl, Jenkins  
> La machine doit avoir au moins **8 Go de RAM** et **4 CPU**.

---

## Étape 1 — Vérifier les outils pré-installés

```bash
git --version
docker --version
minikube version
kubectl version --client
```

Vérifier que Docker fonctionne sans sudo :

```bash
docker ps
```

> Si « permission denied » :
> ```bash
> sudo usermod -aG docker $USER
> newgrp docker
> ```

> 📸 **CAPTURE :** Versions de tous les outils affichées
>
> ![Outils](image/capture_outils.png)

---

## Étape 2 — Cloner le dépôt et se placer sur la branche

```bash
cd ~
git clone https://github.com/LittleWolf-Code/MiniProjet_CaaS.git
cd MiniProjet_CaaS
git checkout projet-final
```

Vérifier la structure :

```bash
ls -la
# Attendu : vote/  result/  worker/  k8s/  jenkins/  Jenkinsfile  Readme.md
```

> 📸 **CAPTURE :** Repository cloné + `ls -la`
>
> ![Structure](image/capture_structure_projet.png)

---

## Étape 3 — Démarrer Minikube

```bash
minikube start --driver=docker --cpus=4 --memory=4096
```

Vérifier :

```bash
kubectl cluster-info
kubectl get nodes
```

> 📸 **CAPTURE :** Minikube démarré + `kubectl get nodes`
>
> ![Minikube](image/capture_minikube_start.png)

---

## Étape 4 — Construire et pousser les images Docker manuellement

> ⚠️ **Remplacez `litlewolf` par votre identifiant Docker Hub si différent.**

```bash
export DOCKERHUB_USER="litlewolf"

# Build des 3 images
docker build -t $DOCKERHUB_USER/vote:latest ./vote
docker build -t $DOCKERHUB_USER/result:latest ./result
docker build -t $DOCKERHUB_USER/worker:latest ./worker
```

Vérifier :

```bash
docker images | grep $DOCKERHUB_USER
```

Pousser vers Docker Hub :

```bash
docker login
# Entrer votre identifiant et mot de passe Docker Hub

docker push $DOCKERHUB_USER/vote:latest
docker push $DOCKERHUB_USER/result:latest
docker push $DOCKERHUB_USER/worker:latest
```

> 📸 **CAPTURE :** `docker images` montrant les 3 images
>
> ![Docker Images](image/capture_docker_images.png)

> 📸 **CAPTURE :** Push réussi vers Docker Hub
>
> ![Docker Push](image/capture_docker_push.png)

---

## Étape 5 — Déployer sur Kubernetes

```bash
# Appliquer tous les manifests d'un coup
kubectl apply -f k8s/
```

Attendre que tout soit prêt :

```bash
kubectl rollout status deployment/vote
kubectl rollout status deployment/result
kubectl rollout status deployment/worker
kubectl rollout status deployment/redis
kubectl rollout status deployment/db
```

Vérifier :

```bash
kubectl get pods
kubectl get svc
kubectl get deploy
```

> Tous les pods doivent être en état **Running** (attendre 1-2 min si nécessaire).

Accéder à l'application :

```bash
minikube service vote --url
# → Ouvrir l'URL dans le navigateur (port 31000)

minikube service result --url
# → Ouvrir l'URL (port 31001)
```

**Tester :** Voter pour Cats ou Dogs → vérifier les résultats en temps réel.

> 📸 **CAPTURE :** `kubectl get pods` — tous Running
>
> ![Pods](image/capture_kubectl_pods.png)

> 📸 **CAPTURE :** `kubectl get svc` + `kubectl get deploy`
>
> ![Services](image/capture_kubectl_services.png)

> 📸 **CAPTURE :** Interface de vote
>
> ![Vote](image/capture_vote_app.png)

> 📸 **CAPTURE :** Résultats en temps réel
>
> ![Result](image/capture_result_app.png)

---

## Étape 6 — Installer Jenkins dockerisé

```bash
# Créer le réseau minikube pour que Jenkins puisse communiquer avec le cluster
docker network create minikube 2>/dev/null || true
```

```bash
# Lancer Jenkins + Docker-in-Docker
cd ~/MiniProjet_CaaS/jenkins
docker compose up -d
```

Vérifier que les conteneurs tournent :

```bash
docker ps | grep jenkins
# Attendu : jenkins-blueocean (Jenkins) + jenkins-docker (DinD)
```

Récupérer le mot de passe admin initial :

```bash
# Attendre ~30 secondes que Jenkins démarre
sleep 30
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

Ouvrir Jenkins dans le navigateur : **http://localhost:8080**

1. Coller le mot de passe admin initial
2. Choisir **« Install suggested plugins »** → attendre l'installation
3. Créer un utilisateur admin (ou continuer avec admin)

> 📸 **CAPTURE :** Jenkins dashboard après installation
>
> ![Jenkins](image/capture_jenkins_dashboard.png)

---

## Étape 7 — Configurer les credentials Docker Hub dans Jenkins

1. Dans Jenkins, aller dans **Manage Jenkins** → **Credentials**
2. Cliquer sur **(global)** → **Add Credentials**
3. Remplir :

| Champ | Valeur |
|-------|--------|
| **Kind** | Username with password |
| **Username** | Votre identifiant Docker Hub |
| **Password** | Votre mot de passe Docker Hub |
| **ID** | `dockerhub-credentials` |
| **Description** | Docker Hub Credentials |

4. Cliquer **Create**

> 📸 **CAPTURE :** Credentials Docker Hub configurés
>
> ![Credentials](image/capture_jenkins_credentials.png)

---

## Étape 8 — Copier la config kubectl dans Jenkins

```bash
# Copier la config kubeconfig dans le conteneur Jenkins
docker cp ~/.kube/config jenkins-blueocean:/home/jenkins/.kube/config

# Corriger les permissions
docker exec -u root jenkins-blueocean chown -R jenkins:jenkins /home/jenkins/.kube
```

Vérifier que Jenkins peut accéder au cluster :

```bash
docker exec jenkins-blueocean kubectl get nodes
# Attendu : le nœud minikube affiché
```

> **Si ça ne marche pas :** le problème est souvent que l'adresse du cluster dans kubeconfig pointe vers `127.0.0.1` mais Jenkins est dans un conteneur Docker. Solution :
>
> ```bash
> # Récupérer l'IP de minikube accessible depuis le réseau Docker
> MINIKUBE_IP=$(minikube ip)
>
> # Remplacer l'adresse dans la config copiée
> docker exec jenkins-blueocean sed -i "s|https://127.0.0.1:[0-9]*|https://$MINIKUBE_IP:8443|g" /home/jenkins/.kube/config
> docker exec jenkins-blueocean sed -i "s|certificate-authority: .*|insecure-skip-tls-verify: true|g" /home/jenkins/.kube/config
>
> # Re-tester
> docker exec jenkins-blueocean kubectl get nodes
> ```

---

## Étape 9 — Créer et lancer la pipeline Jenkins

1. Dans Jenkins, cliquer sur **New Item**
2. Nom : `MiniProjet-CaaS`
3. Type : **Pipeline**
4. Configuration :
   - **Pipeline** → **Definition** : `Pipeline script from SCM`
   - **SCM** : Git
   - **Repository URL** : `https://github.com/LittleWolf-Code/MiniProjet_CaaS.git`
   - **Branch** : `*/projet-final`
   - **Script Path** : `Jenkinsfile`
5. Cliquer **Save**

6. Cliquer **Build Now** → observer les 4 stages :
   - ✅ Checkout
   - ✅ Build Docker Images
   - ✅ Push to Docker Hub
   - ✅ Deploy to Kubernetes

> 📸 **CAPTURE :** Pipeline Jenkins — 4 stages réussis (Stage View)
>
> ![Pipeline](image/capture_jenkins_pipeline.png)

> 📸 **CAPTURE :** Console Output — SUCCESS
>
> ![Console](image/capture_jenkins_console.png)

---

## Étape 10 — Installer Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Étape 11 — Installer le monitoring (Prometheus + Grafana)

```bash
# Ajouter le repo Helm de Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Installer la stack complète
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin
```

Attendre que tous les pods soient prêts (~2-3 minutes) :

```bash
kubectl get pods -n monitoring
# Attendre que tout soit Running (relancer si nécessaire)
```

---

## Étape 12 — Accéder à Grafana

```bash
# Exposer Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
```

Ouvrir : **http://localhost:3000**

| Champ | Valeur |
|-------|--------|
| **Login** | `admin` |
| **Mot de passe** | `admin` |

### Importer un dashboard

1. **Dashboards** → **Import**
2. Entrer l'ID : **`15661`** → **Load** → sélectionner datasource **Prometheus** → **Import**

Dashboards recommandés :

| ID | Dashboard |
|----|---------:|
| **15661** | Kubernetes Cluster Monitoring |
| **6417** | Kubernetes Pods Monitoring |
| **315** | Kubernetes Cluster Overview |

> 📸 **CAPTURE :** Dashboard Grafana avec métriques CPU/mémoire
>
> ![Grafana](image/capture_grafana_dashboard.png)

### Accéder à Prometheus (optionnel)

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
```

Ouvrir : **http://localhost:9090**

Requêtes utiles :

```promql
# CPU par pod
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Mémoire par pod
sum(container_memory_usage_bytes{namespace="default"}) by (pod)
```

> 📸 **CAPTURE :** Prometheus avec requête exécutée
>
> ![Prometheus](image/capture_prometheus.png)

---

## 🌐 Résumé des ports et accès

| Service | Type | Port | URL |
|---------|------|:----:|-----|
| **Vote** | NodePort | 31000 | `http://<MINIKUBE_IP>:31000` |
| **Result** | NodePort | 31001 | `http://<MINIKUBE_IP>:31001` |
| **Redis** | ClusterIP | 6379 | Interne |
| **PostgreSQL** | ClusterIP | 5432 | Interne |
| **Jenkins** | Docker | 8080 | `http://localhost:8080` |
| **Grafana** | Port-forward | 3000 | `http://localhost:3000` |
| **Prometheus** | Port-forward | 9090 | `http://localhost:9090` |

```bash
minikube ip    # Récupérer l'IP de Minikube
```

---

## 🔧 Dépannage

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

### Jenkins ne peut pas accéder à Docker (DinD)

```bash
# Vérifier que le conteneur DinD tourne
docker ps | grep jenkins-docker

# Si non, relancer :
cd ~/MiniProjet_CaaS/jenkins
docker compose down
docker compose up -d
```

### Jenkins ne peut pas accéder à Kubernetes

```bash
# Re-copier la config kubectl
docker cp ~/.kube/config jenkins-blueocean:/home/jenkins/.kube/config
docker exec -u root jenkins-blueocean chown -R jenkins:jenkins /home/jenkins/.kube

# Adapter l'IP si nécessaire
MINIKUBE_IP=$(minikube ip)
docker exec jenkins-blueocean sed -i "s|https://127.0.0.1:[0-9]*|https://$MINIKUBE_IP:8443|g" /home/jenkins/.kube/config
docker exec jenkins-blueocean sed -i "s|certificate-authority: .*|insecure-skip-tls-verify: true|g" /home/jenkins/.kube/config
```

### Réinitialiser tout le déploiement

```bash
kubectl delete -f k8s/
kubectl apply -f k8s/
```

### Relancer le monitoring

```bash
helm uninstall monitoring -n monitoring
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin
```

---

## 🎯 Conclusion

Ce projet démontre la mise en place d'une **chaîne DevOps complète** :

| Étape | Réalisation |
|-------|------------|
| **Code source** | ✅ Repository GitHub avec 3 microservices structurés |
| **Dockerisation** | ✅ 3 images Docker construites et poussées sur Docker Hub |
| **CI/CD** | ✅ Pipeline Jenkins (Checkout → Build → Push → Deploy) |
| **Orchestration** | ✅ Kubernetes avec 5 services sur Minikube |
| **Monitoring** | ✅ Prometheus + Grafana via Helm |

**Technologies :** Git, GitHub, Docker, Docker Hub, Jenkins, Kubernetes (Minikube), Helm, Prometheus, Grafana

---

> **Auteur :** Lopvet Lucas  
> **Module :** Cloud as a Service (CaaS)  
> **Date :** 12/02/2026