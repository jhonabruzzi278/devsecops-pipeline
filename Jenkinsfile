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

        stage('SCA - pip-audit') {
            steps {
                sh '''
                    mkdir -p reports
                    docker run --rm --volumes-from jenkins -w "$WORKSPACE" python:3.12-slim sh -c '
                        pip install -q pip-audit
                        pip-audit -r requirements.txt --no-deps --disable-pip > reports/pip-audit.txt 2>&1
                        RC=$?
                        cat reports/pip-audit.txt
                        exit $RC
                    '
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    mkdir -p reports
                    RC=0
                    docker rm -f zap-${BUILD_NUMBER} || true
                    docker run --name zap-${BUILD_NUMBER} --user root --network ${NETWORK} \
                        -v zap-wrk-${BUILD_NUMBER}:/zap/wrk \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py -t http://${APP_NAME}:5000/ -r zap_report.html -J zap_report.json || RC=$?
                    docker cp zap-${BUILD_NUMBER}:/zap/wrk/. reports/ || true
                    docker rm zap-${BUILD_NUMBER}
                    docker volume rm zap-wrk-${BUILD_NUMBER} || true
                    echo "ZAP terminó con código $RC (0 = sin alertas, 1 = fallos, 2 = advertencias). Esta etapa informa, no bloquea."
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
        success { echo "Pipeline OK: http://localhost:5000" }
        failure { echo 'Pipeline falló: abre la etapa en rojo y lee el mensaje.' }
    }
}
