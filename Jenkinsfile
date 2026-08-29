pipeline {
    agent any

    environment {
        REGISTRY = 'docker-registry:5000'
        REGISTRY_CREDENTIALS_ID = 'docker-credentials'
        GIT_CREDENTIALS_ID = 'github-credentials'
        IMAGE_NAME = 'faang-account-service'
        OVERLAY_FILE = 'k8s/overlays/homelab/kustomization.yaml'
    }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds(abortPrevious: true)
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule sync --recursive'
                sh 'git submodule update --init --recursive'
                script {
                    env.IMAGE_TAG = sh(
                        returnStdout: true,
                        script: 'git -C faang-account_service rev-parse --short=12 HEAD'
                    ).trim()
                }
            }
        }

        stage('Build and Test') {
            steps {
                dir('faang-account_service') {
                    sh './gradlew clean build --no-daemon --stacktrace'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'faang-account_service/build/test-results/test/*.xml'
                }
                success {
                    archiveArtifacts artifacts: 'faang-account_service/build/libs/service.jar', fingerprint: true
                }
            }
        }

        stage('Build Image') {
            when {
                allOf {
                    not { changeRequest() }
                    anyOf {
                        branch 'dev-local'
                        environment name: 'GIT_BRANCH', value: 'origin/dev-local'
                    }
                }
            }
            steps {
                dir('faang-account_service') {
                    sh 'docker build --pull --tag "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG" .'
                }
            }
        }

        stage('Publish Image') {
            when {
                allOf {
                    not { changeRequest() }
                    anyOf {
                        branch 'dev-local'
                        environment name: 'GIT_BRANCH', value: 'origin/dev-local'
                    }
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.REGISTRY_CREDENTIALS_ID,
                    usernameVariable: 'REGISTRY_USERNAME',
                    passwordVariable: 'REGISTRY_PASSWORD'
                )]) {
                    sh '''
                        set +x
                        echo "$REGISTRY_PASSWORD" | docker login "$REGISTRY" --username "$REGISTRY_USERNAME" --password-stdin
                        docker push "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG"
                        docker logout "$REGISTRY"
                    '''
                }
            }
        }

        stage('Update GitOps Image') {
            when {
                allOf {
                    not { changeRequest() }
                    anyOf {
                        branch 'dev-local'
                        environment name: 'GIT_BRANCH', value: 'origin/dev-local'
                    }
                }
            }
            steps {
                dir('faang-infra') {
                    sh 'git fetch origin dev-local'
                    sh 'git checkout -B dev-local origin/dev-local'
                    sh 'sh ops/jenkins/update-image-tag.sh "$OVERLAY_FILE" "$REGISTRY/$IMAGE_NAME" "$IMAGE_TAG"'
                    sh 'kubectl kustomize k8s/overlays/homelab > /dev/null'
                    withCredentials([gitUsernamePassword(
                        credentialsId: env.GIT_CREDENTIALS_ID,
                        gitToolName: 'Default'
                    )]) {
                        sh '''
                            if git diff --quiet -- "$OVERLAY_FILE"; then
                                echo "Image tag is already $IMAGE_TAG; no GitOps commit is needed."
                                exit 0
                            fi
                            git config user.name "Jenkins"
                            git config user.email "jenkins@faang.local"
                            git add "$OVERLAY_FILE"
                            git commit -m "deploy(account): $IMAGE_TAG"
                            git push origin HEAD:dev-local
                        '''
                    }
                }
            }
        }
    }

    post {
        cleanup {
            sh 'docker image rm "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG" >/dev/null 2>&1 || true'
            deleteDir()
        }
    }
}
