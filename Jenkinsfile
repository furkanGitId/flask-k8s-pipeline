pipeline {
    agent any

    environment {
        DOCKER_IMAGE  = "my-app"
        DOCKER_TAG    = "${BUILD_NUMBER}"
        KUBECONFIG    = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('📥 Clone Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/furkanGitId/flask-k8s-pipeline.git'
            }
        }

        stage('🔍 Verify Tools') {
            steps {
                sh 'docker --version'
                sh 'kubectl version --client'
                sh 'minikube version'
            }
        }

        stage('🐳 Build Docker Image inside Minikube') {
            steps {
                // Point Docker CLI to Minikube's internal Docker daemon
                // So the image is built INSIDE Minikube — no push needed
                sh '''
                    # Load minikube docker env properly using eval
                    eval $(minikube -p minikube docker-env)

                    docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
                    docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest
                    echo "✅ Image built: $DOCKER_IMAGE:$DOCKER_TAG"
                '''
            }
        }

        stage('🚀 Deploy to Kubernetes') {
            steps {
                sh """
                    # Replace IMAGE_TAG placeholder in deployment yaml with actual image name
                    sed -i 's|IMAGE_TAG|${DOCKER_IMAGE}:${DOCKER_TAG}|g' k8s/deployment.yaml
                    
                    # Apply Kubernetes manifests
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    # Wait until rollout finishes successfully
                    kubectl rollout status deployment/my-app
                """
            }
        }

        stage('✅ Show Running Pods') {
            steps {
                // Print the URL where your app is accessible on Ubuntu
                sh """
                    echo "--- Running Pods ---"
                    kubectl get pods -l app=my-app

                    echo "--- Service Info ---"
                    kubectl get service my-app-service

                    echo "--- App URL ---"
                    minikube service my-app-service --url
                """
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline PASSED — App is live on Kubernetes!'
        }
        failure {
            echo '❌ Pipeline FAILED — Check stage logs above.'
            // Show pod logs to help debug
            sh 'kubectl describe pods -l app=my-app || true'
        }
    }
}