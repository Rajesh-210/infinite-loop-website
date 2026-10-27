pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        IMAGE_NAME = 'infinite-loop-website'
        CONTAINER_NAME = 'infinite-loop-website'
        HOST_PORT = '8081'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Gitleaks Secret Scan') {
            steps {
                sh '''
                    set +e
                    mkdir -p reports

                    gitleaks detect \
                        --source . \
                        --redact \
                        --report-format json \
                        --report-path reports/gitleaks-report.json
                    SCAN_STATUS=$?
                    set -e

                    exit $SCAN_STATUS
                ''' 
            }
        }   

        stage('Validate Project') {
            steps {
                sh '''
                    set -eu
                    test -f index.html
                    test -f Dockerfile
                    test -d css
                    test -d js
                    test -d img
                    echo "Project structure validation successful."
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    script {
                        def scannerHome = tool 'SonarScanner'
                        sh '''
                            set -eu
                            "${scannerHome}/bin/sonar-scanner"
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    set -eu

                    echo "Checking Trivy installation..."
                    trivy --version

                    echo "Creating security report..."

                    mkdir -p reports

                    trivy image \
                        --scanners vuln \
                        --format table \
                        --output reports/trivy-report.txt \
                        ${IMAGE_NAME}:${BUILD_NUMBER}
                    
                    echo "Enforcing HIGH and CRITICAL vulnerability policy..."    

                    echo "Scanning image: ${IMAGE_NAME}:${BUILD_NUMBER}"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Trivy scan passed. No HIGH or CRITICAL vulnerabilities found."
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                sh '''
                    set -eu

                    TEST_CONTAINER="infinite-loop-test-${BUILD_NUMBER}"

                    cleanup() {
                        docker rm -f "$TEST_CONTAINER" >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT

                    docker run -d \
                        --name "$TEST_CONTAINER" \
                        -p 127.0.0.1::80 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    HOST_PORT_TEST=$(docker port "$TEST_CONTAINER" 80/tcp | sed 's/.*://')

                    for i in $(seq 1 20); do
                        if curl -fsS "http://127.0.0.1:${HOST_PORT_TEST}/" -o /dev/null; then
                            echo "Website HTTP test passed."
                            exit 0
                        fi
                        sleep 2
                    done

                    echo "Website HTTP test failed."
                    exit 1
                '''
            }
        }

        stage('Deploy Website') {
            steps {
                sh '''
                    set -eu

                    docker rm -f ${CONTAINER_NAME} >/dev/null 2>&1 || true

                    docker run -d \
                        --restart unless-stopped \
                        --name ${CONTAINER_NAME} \
                        -p ${HOST_PORT}:80 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Website deployed successfully."
                '''
            }
        }
    }

    post {

        always {
            archiveArtifacts artifacts: 'reports/trivy-report.txt,reports/gitleaks-report.json',
                allowEmptyArchive: true
            echo 'Trivy security report archiving step completed.'
        }
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }
    }
}
