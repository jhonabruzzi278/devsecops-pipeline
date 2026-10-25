pipeline {
    agent any

    environment {
        IMAGE_NAME = 'flask-vuln-app'
        APP_NAME   = 'flask-app'
        NETWORK    = 'devsecops-net'
    }

    stages {
        stage('Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Test') {
            steps {
                sh 'docker run --rm ${IMAGE_NAME}:${BUILD_NUMBER} sh -c "pip install -q pytest && python -m pytest -q tests"'
            }
        }

        stage('SAST - Bandit') {
            steps {
                sh '''
                    mkdir -p reports
                    docker run --rm --volumes-from jenkins -w "$WORKSPACE" python:3.12-slim sh -c '
                        pip install -q bandit
                        bandit -r app.py -f txt -o reports/bandit.txt || true
                        cat reports/bandit.txt
                        bandit -r app.py -lll -q > /dev/null || true
                    '
                '''
            }
        }

        stage('SAST - SonarQube') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        docker run --rm --user root --network ${NETWORK} \
                            --volumes-from jenkins -w "$WORKSPACE" \
                            -e SONAR_HOST_URL=http://sonarqube:9000 \
                            -e SONAR_TOKEN="$SONAR_TOKEN" \
                            sonarsource/sonar-scanner-cli:latest \
                            -Dsonar.projectKey=devsecops-lab \
                            -Dsonar.sources=. -Dsonar.exclusions=tests/**,reports/**
                        echo "Resultados: http://localhost:9000/dashboard?id=devsecops-lab"
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker network create ${NETWORK} || true
                    docker rm -f ${APP_NAME} || true
                    docker run -d --name ${APP_NAME} --network ${NETWORK} -p 5000:5000 ${IMAGE_NAME}:${BUILD_NUMBER}
                    for i in 1 2 3 4 5 6 7 8 9 10; do
                        docker exec ${APP_NAME} python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')" && exit 0
                        sleep 2
                    done
                    echo "La app no respondió"; exit 1
                '''
            }
        }

        // >>> AQUÍ agregarás la etapa SCA (paso 7) y la etapa DAST (paso 8)
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
        success { echo "Pipeline OK: http://localhost:5000" }
        failure { echo 'Pipeline falló: abre la etapa en rojo y lee el mensaje.' }
    }
}
