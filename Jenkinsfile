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
                    set -e
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v $PWD:/workspace \
                      -w /workspace \
                      aquasec/trivy:0.51.4 image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --no-progress \
                        --format table \
                        --output /workspace/trivy-report.txt \
                        $IMAGE:$TAG

                    cat trivy-report.txt
                '''
                archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('', REGISTRY_CREDENTIALS) {
                        sh 'docker push $IMAGE:$TAG'
                        sh 'docker push $IMAGE:latest'
                    }
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

