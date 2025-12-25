pipeline {
    agent any

    environment {
        IMAGE                = 'trrkos1/devportfolio'
        REGISTRY_CREDENTIALS = 'docker-hub'
        TAG                  = "${BUILD_NUMBER}" // будет переопределен в Checkout
    }

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def shortSha = env.GIT_COMMIT?.take(8)
                    if (!shortSha?.trim()) {
                        shortSha = sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
                    }
                    env.TAG = "${env.BUILD_NUMBER}-${shortSha}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t $IMAGE:$TAG -t $IMAGE:latest .'
            }
        }

        stage('Scan') {
            steps {
                sh '''
                    set -eo pipefail
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:0.51.4 image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --no-progress \
                        --format table \
                        $IMAGE:$TAG | tee trivy-report.txt
                '''
                archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: REGISTRY_CREDENTIALS, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
                        docker push $IMAGE:$TAG
                        docker push $IMAGE:latest
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f'
        }
    }
}

