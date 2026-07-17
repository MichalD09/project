pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'twoj-login-dockerhub/simple-cicd-app'
        DOCKER_CREDENTIALS_ID = 'dockerhub-token'
        CONTAINER_NAME = 'simple-cicd-app-test'
        APP_PORT = '5000'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Pobieram kod z repozytorium...'
                checkout scm

                script {
                    env.COMMIT_HASH = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${env.DOCKER_IMAGE}:${env.COMMIT_HASH}"

                    echo "Tag obrazu: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Test') {
            steps {
                echo 'Instalacja zaleznosci i uruchomienie testow...'

                sh '''
                    pip install -r requirements.txt
                    pytest
                '''
            }
        }

        stage('Build') {
            steps {
                echo 'Budowa obrazu Docker...'

                sh '''
                    docker build -t "$IMAGE_TAG" .
                '''
            }
        }

        stage('Push') {
            steps {
                echo 'Publikacja obrazu do Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$IMAGE_TAG"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy & Test') {
            steps {
                echo 'Uruchomienie kontenera i smoke-test...'

                sh '''
                    docker rm -f "$CONTAINER_NAME" || true

                    docker run -d \
                        --name "$CONTAINER_NAME" \
                        -p "$APP_PORT:5000" \
                        "$IMAGE_TAG"

                    sleep 5

                    curl -f "http://localhost:$APP_PORT/health"
                '''
            }
        }
    }

    post {
        always {
            echo 'Sprzatanie po pipeline...'

            sh '''
                docker rm -f "$CONTAINER_NAME" || true
            '''
        }

        success {
            echo 'Pipeline zakonczony sukcesem.'
        }

        failure {
            echo 'Pipeline zakonczony bledem.'
        }
    }
}
