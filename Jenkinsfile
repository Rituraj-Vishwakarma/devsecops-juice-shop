pipeline {
    agent any

    environment {
        REPORTS_DIR = "reports"
        APP_NAME = "juice-shop"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Rituraj-Vishwakarma/devsecops-juice-shop.git'
                sh 'mkdir -p ${REPORTS_DIR}'
            }
        }

        stage('Secret Scanning - Gitleaks') {
            steps {
                sh '''
                  docker run --rm -v $(pwd):/path zricethezav/gitleaks \
                    detect --source /path --report-path /path/${REPORTS_DIR}/gitleaks.json --no-git
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORTS_DIR}/gitleaks.json", fingerprint: true
                }
            }
        }

        stage('SAST - Semgrep') {
            steps {
                sh '''
                  docker run --rm -v $(pwd):/src returntocorp/semgrep semgrep \
                    --config "p/owasp-top-ten" --json > ${REPORTS_DIR}/semgrep.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORTS_DIR}/semgrep.json", fingerprint: true
                }
            }
        }

        stage('SCA - Trivy FS') {
            steps {
                sh '''
                  docker run --rm -v $(pwd):/src aquasec/trivy fs /src \
                    --format json --output ${REPORTS_DIR}/trivy-fs.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORTS_DIR}/trivy-fs.json", fingerprint: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${APP_NAME}:latest .'
            }
        }

        stage('Container Scan - Trivy Image') {
            steps {
                sh '''
                  docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy \
                    image ${APP_NAME}:latest --format json --output ${REPORTS_DIR}/trivy-image.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORTS_DIR}/trivy-image.json", fingerprint: true
                }
            }
        }

        stage('Deploy App for DAST') {
            steps {
                sh 'docker run -d --rm --name ${APP_NAME}-test -p 3000:3000 ${APP_NAME}:latest'
                sleep 15 // give app time to start
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                  docker run --rm --network="host" owasp/zap2docker-stable zap-baseline.py \
                    -t http://localhost:3000 -r ${REPORTS_DIR}/zap_report.html || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORTS_DIR}/zap_report.html", fingerprint: true
                }
            }
        }

        stage('Clean Up') {
            steps {
                sh 'docker stop ${APP_NAME}-test || true'
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Reports archived in Jenkins."
        }
    }
}
