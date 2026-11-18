pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://sonarqube:9000'
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {
        stage('Build') {
            steps {
                echo 'Preparando y validando código fuente Flask...'
                sh 'ls -la'
            }
        }

        stage('Test') {
            steps {
                echo 'Ejecutando pruebas básicas de sintaxis y archivos...'
                sh 'test -f app.py && test -f requirements.txt'
            }
        }

        stage('Analyze') {
            steps {
                echo 'Ejecutando análisis estático SAST con SonarQube...'
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=devsecops-flask-app -Dsonar.sources=. -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_TOKEN}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Desplegando aplicación Flask en contenedor de prueba...'
                sh '''
                    docker rm -f flask-vulnerable-app || true
                    docker create --name flask-vulnerable-app --network devsecops-net -p 5000:5000 -w /app python:3.9-slim bash -c "pip install -r requirements.txt && python app.py"
                    docker cp . flask-vulnerable-app:/app
                    docker start flask-vulnerable-app
                    sleep 15
                '''
            }
        }

        stage('Security Test') {
            steps {
                echo '1. Ejecutando análisis de dependencias (SCA) con OWASP Dependency-Check...'
                dependencyCheck additionalArguments: '--scan . --nvdApiKey 40B1DD46-99E3-F111-8379-129478FCB64D --format HTML --format XML --out .', odcInstallation: 'DependencyCheck'

                echo '2. Ejecutando análisis dinámico (DAST) con OWASP ZAP...'
                sh '''
                    rm -rf zap_report && mkdir -p zap_report
                    docker volume rm -f zap_wrk || true
                    docker run --name zap-scanner --network devsecops-net -u root -v zap_wrk:/zap/wrk ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t "http://flask-vulnerable-app:5000/hello?name=TEST" -r zap_report.html || true
                    docker cp zap-scanner:/zap/wrk/zap_report.html zap_report/zap_report.html
                    docker rm -f zap-scanner || true
                    docker volume rm -f zap_wrk || true
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'dependency-check-report.html, zap_report/zap_report.html', allowEmptyArchive: true
        }
    }
}
