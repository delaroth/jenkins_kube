pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = 'delaroth'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/my-kubernetes-app"
    }

    stages {
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
                        // Note: The warning about insecure Groovy string interpolation for DOCKER_PASSWORD
                        // is often acceptable here since it's piped to stdin.
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
                    withCredentials([string(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
                        sh """
                            TEMP_KUBECONFIG_FILE=\$(mktemp)
                            # Use printf %s to write the content exactly as it is, without interpretation
                            printf '%s' "${KUBECONFIG_CONTENT}" > "\$TEMP_KUBECONFIG_FILE"
                            chmod 600 "\$TEMP_KUBECONFIG_FILE" # Set appropriate permissions

                            echo "Applying Kubernetes manifests and updating deployment..."

                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl apply -f kubernetes/deployment.yaml
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl apply -f kubernetes/service.yaml
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl set image deployment/my-kubernetes-app-deployment my-kubernetes-app=${IMAGE_NAME}:${env.BUILD_NUMBER}

                            echo "Deployment updated to use image: ${IMAGE_NAME}:${env.BUILD_NUMBER}"

                            rm "\$TEMP_KUBECONFIG_FILE"
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_CONTENT_VERIFY')]) {
                        sh """
                            TEMP_KUBECONFIG_FILE=\$(mktemp)
                            # Use printf %s here as well
                            printf '%s' "${KUBECONFIG_CONTENT_VERIFY}" > "\$TEMP_KUBECONFIG_FILE"
                            chmod 600 "\$TEMP_KUBECONFIG_FILE"

                            echo "Waiting for deployment rollout to complete..."
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl rollout status deployment/my-kubernetes-app-deployment --timeout=5m
                            echo "Deployment rollout complete."

                            rm "\$TEMP_KUBECONFIG_FILE"
                        """
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
