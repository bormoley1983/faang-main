pipeline {
    agent any

    environment {
        REGISTRY = 'your-registry'
        IMAGE_NAME = 'faang-account-service'
        TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Gradle') {
            steps {
                dir('faang-account_service') {
                    sh './gradlew clean build -x test' 
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('faang-account_service') {
                    sh "docker build -t ${REGISTRY}/${IMAGE_NAME}:${TAG} ."
                    sh "docker tag ${REGISTRY}/${IMAGE_NAME}:${TAG} ${REGISTRY}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${REGISTRY}/${IMAGE_NAME}:${TAG}"
                sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
            }
        }

        stage('Deploy to K8s') {
            steps {
                // Assuming you have kubeconfig set up in Jenkins or use a plugin
                // This is a placeholder for the deployment step
                sh "kubectl apply -f faang-infra/k8s/services/account-service.yaml"
                sh "kubectl set image deployment/faang-account-service account-service=${REGISTRY}/${IMAGE_NAME}:${TAG}"
            }
        }
    }
}
