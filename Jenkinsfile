pipeline {
    agent any // The pipeline will run on any available Jenkins agent (your main Jenkins server)

    environment {
        // Replace with your Docker Hub username
        DOCKER_HUB_USERNAME = 'delaroth' // Example: 'delaroth'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/my-kubernetes-app"
        // KUBECONFIG_PATH is now handled dynamically by withCredentials
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Assuming your Jenkins project is configured to pull from your GitHub repo
                // This step ensures the latest code is in the workspace
                git branch: 'main', url: 'https://github.com/your-username/my-kubernetes-app.git' // Replace with your actual GitHub repo URL
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image, tagging it with the current build number
                    // This creates a unique version for each build
                    sh "docker build -t ${IMAGE_NAME}:${env.BUILD_NUMBER} ."
                    // Also tag with 'latest' for easy reference (though unique tags are better for rollbacks)
                    sh "docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    // Use the Docker Hub credentials (ID 'DockerHub') that you have set up in Jenkins
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        // Log in to Docker Hub using the credentials
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                        // Push the uniquely tagged image
                        sh "docker push ${IMAGE_NAME}:${env.BUILD_NUMBER}"
                        // Push the 'latest' tagged image
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Use 'withCredentials' to retrieve the K3s kubeconfig content from the 'Secret text' credential
                    // The content will be placed in the environment variable 'KUBECONFIG_CONTENT'
                    withCredentials([string(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
                        // Create a temporary file on the Jenkins agent and write the kubeconfig content into it
                        // 'mktemp' creates a unique temporary file path
                        sh """
                            TEMP_KUBECONFIG_FILE=\$(mktemp)
                            echo "${KUBECONFIG_CONTENT}" > "\$TEMP_KUBECONFIG_FILE"
                            chmod 600 "\$TEMP_KUBECONFIG_FILE" # Set appropriate permissions for the temporary file

                            echo "Applying Kubernetes manifests and updating deployment..."

                            // Set KUBECONFIG to the temporary file's path for all kubectl commands in this block
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl apply -f kubernetes/deployment.yaml
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl apply -f kubernetes/service.yaml

                            // Update the deployment's image to the newly built one, triggering a rolling update
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl set image deployment/my-kubernetes-app-deployment my-kubernetes-app=${IMAGE_NAME}:${env.BUILD_NUMBER}

                            echo "Deployment updated to use image: ${IMAGE_NAME}:${env.BUILD_NUMBER}"

                            // Clean up the temporary kubeconfig file for security reasons
                            rm "\$TEMP_KUBECONFIG_FILE"
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    // Repeat the 'withCredentials' and temporary file creation to use kubeconfig for verification
                    withCredentials([string(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_CONTENT_VERIFY')]) {
                        sh """
                            TEMP_KUBECONFIG_FILE=\$(mktemp)
                            echo "${KUBECONFIG_CONTENT_VERIFY}" > "\$TEMP_KUBECONFIG_FILE"
                            chmod 600 "\$TEMP_KUBECONFIG_FILE"

                            echo "Waiting for deployment rollout to complete..."
                            // Use the temporary kubeconfig file to check the rollout status
                            KUBECONFIG=\${TEMP_KUBECONFIG_FILE} kubectl rollout status deployment/my-kubernetes-app-deployment --timeout=5m
                            echo "Deployment rollout complete."

                            // Clean up the temporary kubeconfig file
                            rm "\$TEMP_KUBECONFIG_FILE"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs() // Always clean up the workspace after the build to save space
            script {
                echo "Pipeline finished. Access your app via NodePort using your host machine's IP address: http://<YOUR_HOST_IP_ADDRESS>:30080"
            }
        }
    }
}
