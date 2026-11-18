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
                echo '1. Ejecutando análisis de dependencias (SCA) con OWASP Dependency-Check / OSV...'
                sh '''
                    docker exec flask-vulnerable-app pip install pip-audit
                    docker exec flask-vulnerable-app pip-audit -r requirements.txt -f json -o /app/sca_results.json || true
                    docker cp flask-vulnerable-app:/app/sca_results.json sca_results.json
                    python3 -c "
import json, html
data = json.load(open('sca_results.json'))
rows = ''
for dep in data.get('dependencies', []):
    for v in dep.get('vulns', []):
        rows += f'<tr><td>{dep[\"name\"]}</td><td>{dep[\"version\"]}</td><td><b>{v[\"id\"]}</b></td><td>{\", \".join(v.get(\"aliases\", []))}</td><td>{html.escape(v.get(\"description\", \"\"))}</td></tr>'
open('dependency-check-report.html', 'w').write(f'''<html><head><meta charset=\"utf-8\"><title>OWASP Dependency-Check / SCA Report</title><style>body{{font-family:Arial;margin:20px}}table{{border-collapse:collapse;width:100%}}th,td{{border:1px solid #ccc;padding:8px;text-align:left}}th{{background:#1b365d;color:#fff}}</style></head><body><h1>Reporte de Análisis de Dependencias (SCA) - devsecops-flask-app</h1><table><tr><th>Paquete</th><th>Versión</th><th>ID Vulnerabilidad</th><th>CVE / Alias</th><th>Descripción</th></tr>{rows}</table></body></html>''')
"
                '''

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
