pipeline {
    agent any // The pipeline will run on any available Jenkins agent (your main Jenkins server)

    environment {
        // Replace with your Docker Hub username
        DOCKER_HUB_USERNAME = 'your-dockerhub-username'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/my-kubernetes-app"
        # The Kubeconfig path inside the container is /var/jenkins_home/.kube/config
        KUBECONFIG_PATH = '/var/jenkins_home/.kube/config'
    }

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image, tagging it with the current build number
                    // You could also use a Git commit hash for tagging: git rev-parse --short HEAD
                    sh "docker build -t ${IMAGE_NAME}:${env.BUILD_NUMBER} ."
                    sh "docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${IMAGE_NAME}:latest" // Also tag with latest
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    // Use the Docker Hub credentials we set up in Jenkins
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                        sh "docker push ${IMAGE_NAME}:${env.BUILD_NUMBER}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Set KUBECONFIG environment variable for kubectl
                    // This tells kubectl where to find your cluster config
                    withEnv(["KUBECONFIG=${KUBECONFIG_PATH}"]) {
                        // Apply the Kubernetes manifests
                        sh "kubectl apply -f kubernetes/deployment.yaml"
                        sh "kubectl apply -f kubernetes/service.yaml"

                        // Update the deployment's image to the newly built one
                        sh "kubectl set image deployment/my-kubernetes-app-deployment my-kubernetes-app=${IMAGE_NAME}:${env.BUILD_NUMBER}"

                        echo "Deployment updated to use image: ${IMAGE_NAME}:${env.BUILD_NUMBER}"
                        echo "Check your deployment status with: kubectl get deploy -w"
                        echo "Access your app via NodePort (Kind/Minikube): http://localhost:30080"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs() // Clean up the workspace after every build
        }
    }
}