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

        stage('Checkout') {
            steps {
                echo '📥 Récupération du code source...'
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                echo '🔨 Construction des images Docker...'
                sh "docker build -t ${VOTE_IMAGE}:${BUILD_TAG} -t ${VOTE_IMAGE}:latest ./vote"
                sh "docker build -t ${RESULT_IMAGE}:${BUILD_TAG} -t ${RESULT_IMAGE}:latest ./result"
                sh "docker build -t ${WORKER_IMAGE}:${BUILD_TAG} -t ${WORKER_IMAGE}:latest ./worker"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo '📤 Push des images vers Docker Hub...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${VOTE_IMAGE}:${BUILD_TAG}"
                    sh "docker push ${VOTE_IMAGE}:latest"
                    sh "docker push ${RESULT_IMAGE}:${BUILD_TAG}"
                    sh "docker push ${RESULT_IMAGE}:latest"
                    sh "docker push ${WORKER_IMAGE}:${BUILD_TAG}"
                    sh "docker push ${WORKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo '🚀 Déploiement sur Kubernetes...'
                sh "kubectl apply -f k8s/"
                sh "kubectl rollout status deployment/vote"
                sh "kubectl rollout status deployment/result"
                sh "kubectl rollout status deployment/worker"
                sh "kubectl rollout status deployment/redis"
                sh "kubectl rollout status deployment/db"
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline terminée avec succès !'
        }
        failure {
            echo '❌ La pipeline a échoué.'
        }
        always {
            sh 'docker logout || true'
        }
    }
}
