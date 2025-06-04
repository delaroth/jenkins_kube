pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = 'delaroth'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/my-kubernetes-app"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/delaroth/jenkins_kube.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${env.BUILD_NUMBER} ."
                    sh "docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
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
                    withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECFG_FILE_PATH')]) {
                        // Set KUBECONFIG environment variable for kubectl to use the temporary file
                        withEnv(["KUBECONFIG=${KUBECFG_FILE_PATH}"]) {
                            echo "Applying Kubernetes manifests and updating deployment..."

                            // kubectl will now use the properly generated temporary file
                            sh "kubectl apply -f kubernetes/deployment.yaml"
                            sh "kubectl apply -f kubernetes/service.yaml"
                            sh "kubectl set image deployment/my-kubernetes-app-deployment my-kubernetes-app=${IMAGE_NAME}:${env.BUILD_NUMBER}"

                            echo "Deployment updated to use image: ${IMAGE_NAME}:${env.BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECFG_FILE_PATH')]) {
                        withEnv(["KUBECONFIG=${KUBECFG_FILE_PATH}"]) {
                            echo "Waiting for deployment rollout to complete..."
                            sh "kubectl rollout status deployment/my-kubernetes-app-deployment --timeout=5m"
                            echo "Deployment rollout complete."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            script {
                echo "Pipeline finished. Access your app via NodePort using your host machine's IP address: http://<YOUR_HOST_IP_ADDRESS>:30080"
            }
        }
    }
}
