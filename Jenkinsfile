pipeline {
    agent any

    environment {
        IMAGE                = 'trrkos1/devportfolio'
        REGISTRY_CREDENTIALS = 'docker-hub'
        TRIVY_IMAGE          = 'aquasec/trivy:0.51.4'
    }

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def shortSha = sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortSha}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    docker build -t "$IMAGE:$IMAGE_TAG" -t "$IMAGE:latest" .
                '''
            }
        }

        stage('Scan') {
            steps {
                sh '''
                    set -eo pipefail
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      "$TRIVY_IMAGE" image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --no-progress \
                        --format table \
                        "$IMAGE:$IMAGE_TAG" | tee trivy-report.txt
                '''
                archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: REGISTRY_CREDENTIALS,
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        set -e
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push "$IMAGE:$IMAGE_TAG"
                        docker push "$IMAGE:latest"
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}