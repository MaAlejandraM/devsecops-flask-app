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
                    docker run -d --name flask-vulnerable-app --network devsecops-net -p 5000:5000 -v $(pwd):/app -w /app python:3.9-slim bash -c "pip install -r requirements.txt && python app.py"
                    sleep 10
                '''
            }
        }

        stage('Security Test') {
            steps {
                echo '1. Ejecutando análisis de dependencias (SCA) con OWASP Dependency-Check...'
                dependencyCheck additionalArguments: '--scan . --format HTML --format XML --out .', odcInstallation: 'DependencyCheck'

                echo '2. Ejecutando análisis dinámico (DAST) con OWASP ZAP...'
                sh '''
                    mkdir -p zap_report
                    chmod 777 zap_report
                    docker run --rm --network devsecops-net -v $(pwd)/zap_report:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t "http://flask-vulnerable-app:5000/hello?name=TEST" -r zap_report.html || true
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
