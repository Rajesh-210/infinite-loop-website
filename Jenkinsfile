pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        IMAGE_NAME     = 'infinite-loop-website'
        CONTAINER_NAME = 'infinite-loop-website'
        HOST_PORT      = '8081'
        WEBSITE_IP     = '15.206.149.132'
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '463771485942'
        ECR_REPOSITORY = 'infinite-loop-website'
        ECR_REGISTRY   = '463771485942.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_IMAGE      = '463771485942.dkr.ecr.ap-south-1.amazonaws.com/infinite-loop-website'
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
                    set -eu
                    mkdir -p reports

                    gitleaks detect \
                        --source . \
                        --redact \
                        --report-format json \
                        --report-path reports/gitleaks-report.json
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
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        withEnv(["SCANNER_HOME=${scannerHome}"]) {
                            sh '''
                                set -eu

                                if [ -z "${SONAR_AUTH_TOKEN:-}" ]; then
                                    echo "ERROR: SONAR_AUTH_TOKEN is unavailable."
                                    exit 1
                                fi

                                "$SCANNER_HOME/bin/sonar-scanner" \
                                    -Dsonar.token="$SONAR_AUTH_TOKEN"
                            '''
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    docker tag \
                        ${IMAGE_NAME}:${BUILD_NUMBER} \
                        ${IMAGE_NAME}:latest

                    echo "Docker image built successfully."
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    set -eu

                    mkdir -p reports

                    trivy --version

                    # Generate a complete vulnerability report.
                    trivy image \
                        --scanners vuln \
                        --format table \
                        --output reports/trivy-report.txt \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    # Fail the pipeline for HIGH or CRITICAL vulnerabilities.
                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Trivy security scan passed."
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                sh '''
                    set -eu

                    TEST_CONTAINER="infinite-loop-test-${BUILD_NUMBER}"

                    cleanup() {
                        docker rm -f "$TEST_CONTAINER" \
                            >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT

                    docker run -d \
                        --name "$TEST_CONTAINER" \
                        -p 127.0.0.1::80 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    TEST_PORT=$(docker port "$TEST_CONTAINER" 80/tcp |
                        sed 's/.*://')

                    for i in $(seq 1 20); do
                        if curl -fsS \
                            "http://127.0.0.1:${TEST_PORT}/" \
                            -o /dev/null; then

                            echo "Temporary Docker image test passed."
                            exit 0
                        fi

                        sleep 2
                    done

                    echo "ERROR: Temporary container health check failed."
                    exit 1
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                sh '''
                    set -eu

                    echo "Checking AWS identity available to Jenkins..."
                    aws sts get-caller-identity

                    echo "Logging in to Amazon ECR..."
                    aws ecr get-login-password --region "$AWS_REGION" | \
                        docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    echo "Tagging versioned image for ECR..."
                    docker tag "${IMAGE_NAME}:${BUILD_NUMBER}" "${ECR_IMAGE}:${BUILD_NUMBER}"
                    docker tag "${IMAGE_NAME}:${BUILD_NUMBER}" "${ECR_IMAGE}:latest"

                    echo "Pushing versioned image..."
                    docker push "${ECR_IMAGE}:${BUILD_NUMBER}"

                    echo "Pushing latest tag..."
                    docker push "${ECR_IMAGE}:latest"

                    echo "Image pushed successfully to Amazon ECR."
                '''
            }
        }

        stage('Deploy and Verify') {
            steps {
                sh '''
                    set -eu

                    NEW_IMAGE="${ECR_IMAGE}:${BUILD_NUMBER}"

                    echo "New image: $NEW_IMAGE"

                    # Remember the image used by the existing container before replacing it.
                    PREVIOUS_IMAGE=$(docker inspect \
                        --format='{{.Config.Image}}' \
                        "$CONTAINER_NAME" 2>/dev/null || true)

                    if [ -n "$PREVIOUS_IMAGE" ]; then
                        echo "Previous image: $PREVIOUS_IMAGE"
                    else
                        echo "No previous container found."
                    fi

                    rollback() {
                        echo "ERROR: Deployment verification failed."
                        echo "Attempting rollback..."

                        docker rm -f "$CONTAINER_NAME" \
                            >/dev/null 2>&1 || true

                        if [ -z "$PREVIOUS_IMAGE" ]; then
                            echo "No previous image available for rollback."
                            return 1
                        fi

                        if ! docker image inspect "$PREVIOUS_IMAGE" \
                            >/dev/null 2>&1; then
                            echo "Previous image is unavailable."
                            return 1
                        fi

                        if ! docker run -d \
                            --restart unless-stopped \
                            --name "$CONTAINER_NAME" \
                            -p "${HOST_PORT}:80" \
                            "$PREVIOUS_IMAGE"; then

                            echo "ERROR: Could not restart previous image."
                            return 1
                        fi

                        echo "Previous container restarted."
                        return 0
                    }

                    # Remove the old container only after saving its image.
                    docker rm -f "$CONTAINER_NAME" \
                        >/dev/null 2>&1 || true

                    # Start the new container.
                    if ! docker run -d \
                        --restart unless-stopped \
                        --name "$CONTAINER_NAME" \
                        -p "${HOST_PORT}:80" \
                        "$NEW_IMAGE"; then

                        rollback || true
                        exit 1
                    fi

                    echo "New container started. Checking application..."

                    HEALTHY=false

                    # Verify the application directly through Docker's host port.
                    for i in $(seq 1 20); do
                        if curl -fsS \
                            "http://127.0.0.1:${HOST_PORT}/" \
                            -o /dev/null; then

                            echo "Direct HTTP health check passed."
                            HEALTHY=true
                            break
                        fi

                        echo "Waiting for application... attempt $i/20"
                        sleep 3
                    done

                    # Verify HTTPS through the existing Nginx reverse proxy.
                    if [ "$HEALTHY" = "true" ]; then
                        HTTPS_HEALTHY=false

                        for i in $(seq 1 10); do
                            if curl -kfsS \
                                --resolve "${WEBSITE_IP}:443:127.0.0.1" \
                                "https://${WEBSITE_IP}/" \
                                -o /dev/null; then

                                echo "HTTPS health check passed."
                                HTTPS_HEALTHY=true
                                break
                            fi

                            echo "Waiting for HTTPS verification..."
                            sleep 3
                        done

                        if [ "$HTTPS_HEALTHY" != "true" ]; then
                            HEALTHY=false
                        fi
                    fi

                    # Restore the previous image if either check fails.
                    if [ "$HEALTHY" != "true" ]; then
                        rollback
                        exit 1
                    fi

                    echo "Deployment and health checks completed successfully."
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'reports/trivy-report.txt,reports/gitleaks-report.json',
                allowEmptyArchive: true
            )

            echo 'Security report archiving completed.'
        }

        success {
            echo 'SUCCESS: Pipeline and deployment verification passed.'
        }

        failure {
            echo 'FAILURE: Pipeline failed. Review the console output.'
        }
    }
}
